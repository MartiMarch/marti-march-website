+++
title = "mTLS for Kubernetes agents"
tags = ["k8s", "kubernetes", "Spring Boot", "agent", "cloud", "Go", "security"]
date = "2025-08-20"
lang = "en"
+++

# Introduction

I'm developing a platform with a service that communicates with several agents. The goal is for the service to be responsible for managing information while the agents are in charge of orchestrating Kubernetes jobs, each of which (the agents) is
located in a K8S cluster. Since I'm developing the MVP, I've decided to put the TLS implementation on hold temporarily. It was a bad decision. Evading TLS certificates ended up being a bigger waste of time than directly implementing mTLS (Mutual
Transport Layer Security). The alternative to using mTLS requires creating a robust alternative system; after some experimentation, I realized it was both cumbersome and insecure. Therefore, if your scenario fits what I'm going to describe in this
article, I recommend implementing mTLS directly.

- [The problem](#the-problem)
- [mTLS](#mtls)
- [mTLS on Kubernetes](#mtls-on-kubernetes)

# The problem

On one side, Spring Boot is the server that manages the agent’s information; on the other side, the Kubernetes Go agents can connect/disconnect. ¿How can the server check if it’s a trusted agent to avoid MitTM attack? ¿How can the agent verify if the
server is the real one? With mutual TLS (mTLS), both ends of the channel must give a valid certificate. This approach enables both identities, server and agent, to be trusted by the other.

When mTLS is implemented:

- The server doesn't accept connections from clients that don't present a certificate signed by a trusted authority.
- The agent verifies that the server also has a valid certificate signed by the same CA.
- Communication between both is encrypted, and both sides can validate the identity safety.

To configure mTLS, you need your own CA or one signed by an external entity, and one certificate for every end that is desired to be protected with mTLS. Before certificate creation, it’s convenient to consider that inside a certificate X.509 some 
fields define the expected behavior for every end.

- {{< inlines/black-inline "Key Usage" >}} an extension that defines what the public key contained in the certificate can do
  - {{< inlines/black-inline "Digital Signature" >}} Allows you to digitally sign data
  - {{< inlines/black-inline "Key Encipherment" >}} It is used for key encryption during the TLS handshake.
- {{< inlines/black-inline "Extended Key Usage (EKU)" >}} List of "extended purposes" that limit what the certificate can be used for
  - {{< inlines/black-inline "TLS Web Server Authentication" >}} Indicates that the certificate can be used by a TLS server (for example, an HTTPS server or a Spring Boot API).
  - {{< inlines/black-inline "TLS Web Client Authentication" >}} Indicates that the certificate can be used by a TLS client (for example, a Go agent that initiates connections).

{{< warning/warning >}}
There are many fields. In mTLS, only the mentioned are relevant.
{{< /warning/warning >}}

# mTLS

#### CA creation

The CA key and certificate must be generated in order to sign the server and agent certificates.

```bash
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt
```

#### Server certificate

1. A private key is generated.

```bash
openssl genrsa -out server.key 2048
```

2. A signing request (*.csr file) is created. The purpose is to ask a CA to sign and generate an authentic digital certificate based on that information.

```bash
openssl req -new -key server.key -out server.csr
```

3. A *.ext extension file is configured to define the certificate capabilities considering the X.509 certificate fields that have been previously discussed.

```text
authorityKeyIdentifier=keyid,issuer             # links the certificate to the CA that signed it
basicConstraints=CA:FALSE                       # this certificate is not a CA
keyUsage = digitalSignature, keyEncipherment    # what cryptographic operations are allowed
extendedKeyUsage = serverAuth                   # specifies the purpose of the certificate
subjectAltName = @alt_names                     # adds the subjectAltName field required by the

[alt_names]                                     # define los valores para el campo subjectAltName
DNS.1 = localhost                               # indicates that the certificate is valid for the localhost domain
```

4. The certificate is signed with the signing request, the CA, the server's private key, and the extensions file.

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt \
  -days 825 -sha256 \
  -extfile server.ext
```

The contents of the certificate can be validated using the following command:

```bash
openssl x509 -in server.crt -text -noout
```

#### mTLS on Spring Boot

To configure a Spring Boot project, you must create a certificate container, using either JKS or PKCS 12. In this specific case, I used the latter. You must include both the server.crt/server.key and the ca.crt in the container.

```bash
openssl pkcs12 -export \
  -inkey server.key \
  -in server.crt \
  -certfile ca.crt \
  -name server \
  -out server.p12
```

```bash
keytool -importcert \
  -file ca.crt \
  -alias ca \
  -keystore truststore.p12 \
  -storetype PKCS12 \
  -storepass 123456 \
  -noprompt
```

In the Gradle build process (or Maven, if applicable), you need to configure how containers are loaded. I've placed the files in src/main/resources. I recommend modifying the .gitignore file so it doesn't upload *.p12 files to the repository.

```gradle
processResources {
    from("src/main/resources") {
        include "server.p12"
    }
}

tasks.processResources {
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```

You also need to configure the application.properties.

```properties
server.port=8080
server.ssl.enabled=true
server.ssl.key-store=classpath:server.p12
server.ssl.key-store-password=${SERVER_SSL_KEY_STORE_PASSWORD}
server.ssl.key-store-type=PKCS12

server.ssl.client-auth=want
server.ssl.trust-store=classpath:truststore.p12
server.ssl.trust-store-password=${SERVER_SSL_TRUST_STORE_PASSWORD}
server.ssl.trust-store-type=PKCS12
```

The final step is to create a configuration class within the project's hex where you specify which endpoints should use mTLS and which fields you want to check in the certificate.

```java
package org.api.adapters.mtls;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.SecurityFilterChain;
import java.util.List;

@Configuration
@EnableWebSecurity
public class Mtls {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                // Solo se exige autenticación mTLS en los endpoints que empiezan con /agent/
                .requestMatchers("/agent/**").authenticated()
                .anyRequest().permitAll()
            )
            .x509(x509 -> x509
                // Extrae el nombre del cliente del campo CN del certificado
                .subjectPrincipalRegex("CN=(.*?)(?:,|$)")
                .userDetailsService(userDetailsService())
            );

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        // Para propósitos de demo, aceptamos cualquier certificado cuyo CN esté presente
        return username -> new User(username, "", List.of());
    }
}
```

{{< inlines/black-inline "subjectPrincipalRegex" >}} -> Allows you to extract the username from the CN field of the certificate. This will be used as the username in the system.

{{< inlines/black-inline "userDetailsService" >}} -> In this example, any client with a valid certificate will be accepted, as long as its CN can be extracted. For a production environment, it would be ideal to validate this username against
a database or authorized list.

{{< inlines/black-inline ".requestMatchers" >}} -> It is specified that only endpoints under /agent/** will require a valid certificate. All other endpoints will not be authenticated by mTLS.

The logic can be complicated depending on the accessible fields of the x509 certificate. For example, it's possible to add custom metadata using extensions defined by OIDs. This would allow additional information (such as roles, environments, or IDs)
to be validated directly in the mTLS configuration. To do this, when creating the certificate, in the extension file definition (server.ext), you could add something like this:

```text
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

# Custom extensions, added to the certificate
1.2.3.4.5.6.7.8.1 = ASN1:UTF8String:fake-agent-k8s-name 
  # ASN1:UTF8String -> Es el tipo de la varaible 
  # 1.2.3.4.5.6.7.8.1 -> Numbers reserved for optional fields

[alt_names]
DNS.1 = localhost
```

```java
@Bean
public UserDetailsService userDetailsService() {
  return username -> {
    // Obtain security context certificate
    var auth = org.springframework.security.core.context.SecurityContextHolder
    .getContext()
    .getAuthentication();
        if (auth instanceof org.springframework.security.web.authentication.preauth.x509.X509AuthenticationToken x509Auth) {
            var cert = (X509Certificate) x509Auth.getCredentials();

            // Read the custom extension (OID)
            byte[] value = cert.getExtensionValue("1.2.3.4.5.6.7.8.1");

            if (value != null) {
                String decoded = decodeExtension(value);
                System.out.println("Custom extension: " + decoded);

                // You can use `decoded` to validate or make decisions
                if (!"fake-agent-k8s-name ".equals(decoded)) {
                    throw new RuntimeException("Unauthorized certificate");
                }
            } else {
                throw new RuntimeException("Extension not found in the certificate");
            }
        }

        // If all goes well, we accept the user.
        return new User(username, "", List.of());
    };
}

private String decodeExtension(byte[] extVal) {
    try {
        // Extensiones X.509 tienen un primer byte de ASN.1 con la longitud, lo ignoramos
        var input = new java.io.ByteArrayInputStream(extVal);
        input.read(); // ignorar primer byte ASN.1 (tipo)
        input.read(); // ignorar longitud

        byte[] bytes = input.readAllBytes();
        return new String(bytes, java.nio.charset.StandardCharsets.UTF_8);
    } catch (Exception e) {
        throw new RuntimeException("Error al leer extensión del certificado", e);
    }
}
```

#### Agents certificates

The agent configuration is almost the same as the server configuration, except for certain fields.

1. The private key is generated.

```bash
openssl genrsa -out agent.key 2048
```

2. The signature request is created.

```bash
openssl req -new -key agent.key -out agent.csr
```

3. The extensions file is defined. In this specific case, the agent certificate must act as both a client and a server (clientAuth and serverAuth) since it receives calls from the server.

```bash
openssl x509 -req \
  -in agent.csr \
  -CA ../ca/ca.crt -CAkey ../ca/ca.key -CAcreateserial \
  -out agent.crt \
  -days 825 -sha256 \
  -extfile agent.ext
```

Upon startup, the agent calls a specific server endpoint to register the information in the database, passing the domain, port, name, etc. The agent must generate the HTTP client with the previously configured certificates, protecting the 
endpoints of interest with mTLS.

```go
func CreateRouter(port string) {
	router := mux.NewRouter()	
	router.HandleFunc("/chunk", func(w http.ResponseWriter, r *http.Request) {
      w.Write([]byte("Chunk received"))
    }).Methods("POST")

	// Reads the Certificate Authority (CA) certificate from a file
	caCert, err := ioutil.ReadFile("resources/ca.crt")
	if err != nil {
		log.Fatalf("Error reading CA cert: %v", err)
	}

	// Create a pool of trusted root certificates
	caCertPool := x509.NewCertPool()
	if ok := caCertPool.AppendCertsFromPEM(caCert); !ok {
		log.Fatal("Failed to append CA cert")
	}

	// Configure TLS with Mandatory Client Authentication (mTLS)
	tlsConfig := &tls.Config{
		ClientCAs:  caCertPool,
		ClientAuth: tls.RequireAndVerifyClientCert,
		MinVersion: tls.VersionTLS12,
	}

	// Create a new HTTPS server with custom TLS settings
	server := &http.Server{
		Addr:      ":" + port,
		Handler:   router,
		TLSConfig: tlsConfig,
	}

	// Starts the HTTPS server, loading the server certificate and private key
	err = server.ListenAndServeTLS("resources/agent.crt", "resources/agent.key")
	if err != nil {
		log.Fatalf("Failed to start server: %v", err)
	}
}
```

# mTLS on Kubernetes

When you're working with hundreds or thousands of microservices deployed on Kubernetes, manually managing certificates and mTLS configurations for each service becomes complex and error-prone. This is where Istio (or a similar tool like Traefik 
or cert-manager) comes in. This service mesh abstracts and automates security, including configuring mTLS between services.

Istio acts as a networking middle layer that sits on top of Kubernetes and:
1. Automatically inject an Envoy proxy as a sidecar into each pod of the services you want to protect.
2. The sidecar proxy intercepts all network traffic to and from the pod, managing the TLS connection and authentication.

This way, microservices don't need to modify their code to use TLS, as Istio automatically encrypts and authenticates connections. As you might think, this means the entire code-level implementation is pointless... Well, that depends on whether the
platform is installed on a cluster with Istio or not. What I'm missing is the ability to configure the certificate or not from configuration files or variables.

Istio uses custom Kubernetes resources to configure traffic security between services. The key CRDs for this are:

- {{< inlines/black-inline "PeerAuthentication" >}} Defines the TLS mutual authentication policy that applies to pods.
- {{< inlines/black-inline "stinationRule" >}} Configures how clients (sidecars) communicate with services, specifying whether they use TLS or not.
- {{< inlines/black-inline "AuthorizationPolicy" >}} Control access based on identities and attributes (more granular).

These configurations allow you to apply granular security policies to all services, a specific namespace, or specific routes.

```yaml
# mTLS for all services
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# mTLS for a namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: production-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT


--- # mTLS for a specific service
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-mtls
  namespace: production
spec:
  host: reviews.production.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL


--- ## mTLS for a specific route
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: permissive
  namespace: production
spec:
  mtls:
    mode: PERMISSIVE
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-mtls-on-admin
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  rules:
    - to:
        - operation:
            paths: ["/admin/*"]
      when:
        # 'request.auth.principal' represents the authenticated identity of the client extracted from the mTLS certificate
        - key: request.auth.principal
          values: ["*"]
```
