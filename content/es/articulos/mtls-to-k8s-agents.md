+++
title = "mTLS para agentes de Kubernetes"
tags = ["k8s", "kubernetes", "Spring Boot", "agent", "cloud", "Go", "security"]
date = "2025-08-20"
lang = "es"
+++

# Introducción

Estoy desarrollando una plataforma con un servicio que se comunica con varios agentes. La idea es que el servicio tenga la responsabilidad de gestionar la información mientras que los agentes se encargan de orquestar jobs de Kubernetes, estando
cada uno de estos (los agentes) en un clúster de K8S. Como estoy desarrollando el PMV, decidí dejar de lado temporalmente la implementación del TLS, craso error. Evadir los certificados para el TLS terminó siendo una pérdida de tiempo aún mayor
que directamente implementar mTLS (mutual Transport Layer Security). La alternativa al uso de mTLS requiere crear un sistema alternativo que sea robusto y, tras experimentar un rato, me di cuenta de que era engorroso y poco seguro. Por eso, si tu
escenario se ajusta a lo que voy a describir en este artículo, te recomiendo implementar mTLS directamente.

- [El problema](#el-problema)
- [mTLS](#mtls)
- [mTLS en Kubernetes](#mtls-en-kubernetes)

# El problema

En un extremo está Spring Boot como servidor gestionando la información del agente y, en el otro, los agentes de Go de k8s que pueden conectarse/desconectarse. ¿Cómo puede saber el servidor que el agente es realmente quien dice ser y que no se trata
de un ataque de MiTM? ¿Cómo puede verificar el agente la autenticidad del servidor? En mutual TLS (mTLS) ambos extremos del canal deben presentar un certificado válido. Esto permite que tanto el servidor como el cliente verifiquen la identidad del otro.

Cuando implementas mTLS:

- El servidor no acepta conexiones de clientes que no presenten un certificado firmado por una autoridad certificadora) confiable.
- El agente verifica que el servidor también tenga un certificado válido y firmado por la misma CA (u otra CA confiable).
- La comunicación entre ambos se cifra, y ambos extremos pueden estar seguros de con quién están hablando.

Para configurar mTLS se necesita una CA propia o firmada por una entidad externa y un certificado por cada extremo que se quiera proteger con mTLS. Aunque, antes de pasar a la creación de certificados, hay que tener en cuenta que dentro del certificado
X.509 hay varios campos que definen el comportamiento esperado de cada nexo.

- {{< inlines/black-inline "Key Usage" >}} extensión que define lo que puede hacer la clave pública contenida en el certificado
  - {{< inlines/black-inline "Digital Signature" >}} Permite firmar digitalmente datos
  - {{< inlines/black-inline "Key Encipherment" >}} Sirve para cifrado de claves durante el handshake TLS
- {{< inlines/black-inline "Extended Key Usage (EKU)" >}} Lista de "propósitos extendidos" que limitan para qué se puede usar el certificado
  - {{< inlines/black-inline "TLS Web Server Authentication" >}} Indica que el certificado puede ser usado por un servidor TLS (por ejemplo, un servidor HTTPS o una API de Spring Boot).
  - {{< inlines/black-inline "TLS Web Client Authentication" >}} Indica que el certificado puede ser usado por un cliente TLS (por ejemplo, un agente en Go que inicia conexiones).

{{< warning/warning >}}
Hay muchos más campos. Para el caso de mTLS los citados son los más importantes
{{< /warning/warning >}}

# mTLS

#### Creación de la CA

Hay que generar la clave y el certificado de la CA para poder firmar los certificados del servidor y el agente.

```bash
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt
```

#### Certificado del servidor

1. Se genera una clave privada.

```bash
openssl genrsa -out server.key 2048
```

2. Se crea una solicitud de firma (archivo *.csr). El propósito es pedirle a una CA que firme y genere un certificado digital auténtico basado en esa información.

```bash
openssl req -new -key server.key -out server.csr
```

3. Se configura un archivo de extensiones server.ext para definir las capacidades del certificado considerando los campos del certificado X.509 que se ha comentado previamente.

```text
authorityKeyIdentifier=keyid,issuer             # vincula el certificado con la CA que lo firmó
basicConstraints=CA:FALSE                       # este certificado no es una CA
keyUsage = digitalSignature, keyEncipherment    # qué operaciones criptográficas se permiten
extendedKeyUsage = serverAuth                   # especifica el propósito del certificado
subjectAltName = @alt_names                     # agrega el campo subjectAltName requerido por los

[alt_names]                                     # define los valores para el campo subjectAltName
DNS.1 = localhost                               # indica que el certificado es válido para el dominio localhost
```

4. Se firma el certificado con la solicitud de firma, la CA, la clave privada del servidor y el archivo de extensiones.

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt \
  -days 825 -sha256 \
  -extfile server.ext
```

Se puede validar el contenido del certificado usando el siguiente comando:

```bash
openssl x509 -in server.crt -text -noout
```

### mTLS en Spring Boot

Para configurar en proyecto de Spring Boot hay que crear contenedor de certificados, ya sea bien usando JKS o PKCS 12. En este caso concreto he utilizado este último. Hay que incluir en el contenedor tanto el server.crt/server.key como el ca.crt.

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

En el proceso de construcción de Gradle (o Maven si fuese el caso) hay que configurar la carga de los contenedores. He ubicado los archivos dentro de src/main/resources, te recomiendo modificar el .gitignore para que no suba los *.p12 al repositorio.

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

También hay que configurar el application.properties.

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

El último paso consiste en crear una clase de configuración dentro de la hexagonal del proyecto donde se especifique que endpoints deben usar mTLS y que campos se quieren revisar del certificado.

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

{{< inlines/black-inline "subjectPrincipalRegex" >}} -> Permite extraer el nombre del usuario del campo CN del certificado. Esto es lo que se usará como nombre de usuario en el sistema.

{{< inlines/black-inline "userDetailsService" >}} -> En este ejemplo, cualquier cliente con un certificado válido será aceptado, siempre que su CN pueda extraerse. Para un entorno productivo, lo ideal sería validar ese username contra una 
base de datos o lista autorizada.

{{< inlines/black-inline ".requestMatchers" >}} -> Se indica que solo los endpoints bajo /agent/** requerirán un certificado válido. El resto de endpoints queda sin autenticación mTLS.

La lógica se puede complicar en función de los campos accesible del certificado x509. Por ejemplo, es posible agregar metadatos personalizados usando extensiones definidas por OID. Esto permitiría validar información adicional (como roles, 
entornos o IDs) directamente en la configuración de mTLS. Par ello, en el momento de creación del certificado, en la definición del archivo de extensión (server.ext), se puede agregar algo como esto:

```text
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

# Extensiones personalizadas, agregadas al certificado 
1.2.3.4.5.6.7.8.1 = ASN1:UTF8String:fake-agent-k8s-name 
  # ASN1:UTF8String -> Es el tipo de la varaible 
  # 1.2.3.4.5.6.7.8.1 -> Números resevados para los campos opcionales

[alt_names]
DNS.1 = localhost
```

```java
@Bean
public UserDetailsService userDetailsService() {
  return username -> {
    // Obtener certificado del contexto de seguridad
    var auth = org.springframework.security.core.context.SecurityContextHolder
    .getContext()
    .getAuthentication();
        if (auth instanceof org.springframework.security.web.authentication.preauth.x509.X509AuthenticationToken x509Auth) {
            var cert = (X509Certificate) x509Auth.getCredentials();

            // Leer la extensión personalizada (OID)
            byte[] value = cert.getExtensionValue("1.2.3.4.5.6.7.8.1");

            if (value != null) {
                String decoded = decodeExtension(value);
                System.out.println("Extensión personalizada: " + decoded);

                // Puedes usar `decoded` para validar o tomar decisiones
                if (!"fake-agent-k8s-name ".equals(decoded)) {
                    throw new RuntimeException("Certificado no autorizado");
                }
            } else {
                throw new RuntimeException("Extensión no encontrada en el certificado");
            }
        }

        // Si todo va bien, aceptamos al usuario
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

#### Certificados de los agentes

La configuración del agente es casi igual que la del servidor, excepto por ciertos campos.

1. Se genera la clave privada.

```bash
openssl genrsa -out agent.key 2048
```

2. Se crea la solicitud de firma.

```bash
openssl req -new -key agent.key -out agent.csr
```

3. Se define el archivo de extensiones. En este caso concreto, el certificado del agente tiene que actuar tanto como cliente como servidor (clientAuth y serverAuth) ya que recibe llamadas desde el servidor.

```bash
openssl x509 -req \
  -in agent.csr \
  -CA ../ca/ca.crt -CAkey ../ca/ca.key -CAcreateserial \
  -out agent.crt \
  -days 825 -sha256 \
  -extfile agent.ext
```

Al arrancar el agente llama a cierto endpoint del servidor para que este último registre la información en la base de datos, pasando el dominio, el puerto, el nombre, etc. El agente tiene que generar el cliente http con los certificados previamente 
configurados, protegiendo los endpoints de interés con mTLS.

```go
func CreateRouter(port string) {
	router := mux.NewRouter()	
	router.HandleFunc("/chunk", func(w http.ResponseWriter, r *http.Request) {
      w.Write([]byte("Chunk received"))
    }).Methods("POST")

	// Lee el certificado de la Autoridad Certificadora (CA) desde archivo
	caCert, err := ioutil.ReadFile("resources/ca.crt")
	if err != nil {
		log.Fatalf("Error reading CA cert: %v", err)
	}

	// Crea un pool (conjunto) de certificados raíz de confianza
	caCertPool := x509.NewCertPool()
	if ok := caCertPool.AppendCertsFromPEM(caCert); !ok {
		log.Fatal("Failed to append CA cert")
	}

	// Configurar TLS con verificación obligatoria de cliente (mTLS)
	tlsConfig := &tls.Config{
		ClientCAs:  caCertPool,
		ClientAuth: tls.RequireAndVerifyClientCert,
		MinVersion: tls.VersionTLS12,
	}

	// Crea un nuevo servidor HTTPS con configuración TLS personalizada
	server := &http.Server{
		Addr:      ":" + port,
		Handler:   router,
		TLSConfig: tlsConfig,
	}

	// Inicia el servidor HTTPS, cargando el certificado y clave privada del servidor
	err = server.ListenAndServeTLS("resources/agent.crt", "resources/agent.key")
	if err != nil {
		log.Fatalf("Failed to start server: %v", err)
	}
}
```

# mTLS en Kubernetes

Cuando trabajas con cientos o miles de microservicios desplegados en Kubernetes, gestionar certificados y configuraciones mTLS manualmente para cada servicio se vuelve complejo y propenso a errores. Aquí es donde entra en juego Istio (o una herramienta
similar como Traefik o cert-manager), un service mesh que abstrae y automatiza la seguridad, incluyendo la configuración de mTLS entre servicios.

Istio actúa como una capa intermedia de red que se instala en Kubernetes y que:
1. Inyecta automáticamente un proxy Envoy como sidecar en cada pod de los servicios que deseas proteger.
2. El proxy sidecar intercepta todo el tráfico de red hacia y desde el pod, gestionando la conexión TLS y la autenticación. 

De esta forma los microservicios no necesitan modificar su código para usar TLS, ya que Istio se encarga de cifrar y autenticar las conexiones automáticamente. Como estarás pensando, esto implica que toda la implementación a nivel de código no tiene
sentido... Bueno, eso depende de si la plataforma se instala en un cluster que cuente con Istio o no. Lo que me falta por programar es la capacidad de configurar el certificado o no a partir de archivos de configuración o variables.

Istio usa recursos personalizados de Kubernetes para configurar la seguridad del tráfico entre servicios. Los principales CRDs para esto son:

- {{< inlines/black-inline "PeerAuthentication" >}} Define la política de autenticación mutua TLS que se aplica a los pods.
- {{< inlines/black-inline "stinationRule" >}} Configura cómo los clientes (sidecars) se comunican con los servicios, especificando si usan TLS o no.
- {{< inlines/black-inline "AuthorizationPolicy" >}} Controla el acceso basado en identidades y atributos (más granular).

Estas configuraciones permiten aplicar políticas de seguridad granulares, bien ya sea a todos los servicios, a un namespace específico y o a rutas concretas.

```yaml
# mTLS para todos los servicios
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# mTLS para un namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: production-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT


--- # mTLS para un servicio concreto
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


--- ## mTLS para una ruta específica
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
        # 'request.auth.principal' representa la identidad autenticada del cliente extraída del certificado mTLS
        - key: request.auth.principal
          values: ["*"]
```
