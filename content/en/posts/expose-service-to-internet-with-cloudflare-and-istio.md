+++
title = "Exposing services to the internet using Cloudflare and Istio"
tags = ["k8s", "istio", "cloudflare", "network"]
date = "2025-09-12"
lang = "en"
+++

# Introduction

I have a curiosity to know who visit my webpages, some of them hosted on GitHub. So, I installed Plausible to get SEO metrics, which implies exposing the services of my self-hosted environment. But, ¿how can a service be exposed to the internet without 
dealing with low-performance dynamic DNS or buying a public IP? This will be explained in this post: how to combine Cloudflare tunnel and Istio to expose the services.

# Domains and DNS

The only thing paid for is the domains. It can be obtained from various domain providers. I have chosen the Cloudflare domain service; domains ending in *.com usually cost around 10 euros per year. 

Once that domain has been purchased, CNAME must be understood; it affects to traffic redirection. CNAME is a DNS record that indicates if one domain is an alias to another domain. When someone is requesting a domain from a browser, DNS redirects the
request to the real "domain" that handles the requests. So, ¿why is this relevant? Because Istio is being used as a reverse proxy, and it needs to expose as many services as possible, Cloudflare requires needs to know where to forward the request.
It brings up a new question: ¿If no static IP will be used, how is it possible that Cloudflare DNS knows where to forward the request? This problem can be bypassed using Cloudflare tunnel. The tunnel is a kubernetes deployment that runs a container which 
opens a socket with a configuration that points to Istio. In other words; Cloudflare works as a reverse proxy that, at the same time, forwards the requests to Istio, another reverse proxy that distributes the traffic among the cluster services.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/tunel.png" >}}

To make the tunnel forward the domain, CNAME must use a wildcard to handle infinite services taking advantage of DNS wildcard, as for example {{< inlines/black-inline "*.fake-domain.com" >}}. In my specific case, I will use {{< inlines/black-inline "pluasible.fake-domain.com" >}}
to point to Plausible service. The CNAME wildcard can be configured from Cloudflare UI:

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/cname.png" >}}

# Istio

Installing Istio in Kubernetes is really easy using a Helm chart.

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingress istio/gateway -n istio-system
```

Istio requires two Kubernetes CRD to forward the requests from the Cloudflare tunnel deployment to the internal cluster services: a Gateway and a VirtualService. The Gateway acts as the traffic entry point to the cluster, configuring the ports, the network protocols, and the
TLS certificates. A VirtualService works as a traffic light. It defines the rules to forward or block the requests (from the Gateway) to the internal service.

The Gateway and VirtualService have been configured to securely expose the Plausible API endpoints, only allowing the sending of metrics from the script installed on the my-fake-webpage.com website. The VirtualService only permits access to the two necessary Plausible 
routes for receiving data (/js and /api/event), routing incoming requests to the internal cluster service. A policy has also been implemented to ensure that only the website's URL can send requests, blocking any other malicious access to the Plausible admin dashboard and 
redirecting unwanted traffic to a fake domain.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: fake-domain-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingress
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        # With the Wildcard you can redirect the Wildcard of the Domain CNAME
        - "*.fake-domain.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  # I'm going to redirect all my traffic on Plausible
  name: plausible
  namespace: istio-system
spec:
  hosts:
    - "plausible.fake-domain.com"
  gateways:
    - fake-domain-gateway
  http:
    - match:
        - uri:
            # Plausible sends metric to the PATH /JS of plausible, only that path is exposed in case any case could try to do log-in the service
            prefix: "/js"
      route:
        - destination:
            # DNS of the Kubernetes Service, points to the clusterip
            host: plausible-http.infra.svc.cluster.local
            port:
              number: 8000
      corsPolicy:
        allowOrigins:
          # I only accept to receive requests from the domain of my website with the plausible script configured
          - exact: https://my-fake-webpage.com
        allowMethods:
          - GET
          - POST
          - OPTIONS
        allowHeaders:
          - Content-Type
        maxAge: 1h
        allowCredentials: false
    - match:
        - uri:
            # This is another endpoint that must be exposed for the sending of metrics
            prefix: "/api/event"
      route:
        - destination:
            host: plausible-http.infra.svc.cluster.local
            port:
              number: 8000
      corsPolicy:
        allowOrigins:
          - exact: https://my-fake-webpage.com
        allowMethods:
          - GET
          - POST
          - OPTIONS
        allowHeaders:
          - Content-Type
        maxAge: 1h
        allowCredentials: false
    # Falls into a failure in case of requesting another path
    - route:
        - destination:
            host: blackhole.svc.cluster.local
```

You can validate whether the request is being redirected correctly with the following curl:

```bash
curl -v -H "Host: fake-domain.com" http://192.168.2.85/{service path}
```

# Cloudflare tunnel

Cloudflare CLI let deploy a tunnel using a pair of commands, have to realize a log-in to obtain JSON credentials. DNS redirection must be configured too.

```bash
# The command will open a window where you must select the purchased domain.
cloudflared tunnel login

# Allows DNS to redirect to Istio
cloudflared tunnel route dns {ID del tunel} fake-domain.com
```

A configuration file must be created to let know to the tunnel the domain/IP destiny. In this case is the Istio Gateway IP.

```yaml
tunnel: {ID del tunel}
# Tunnel credentials
credentials-file: /home/{username}/.cloudflared/{ID del tunel}.json

ingress:
  - hostname: fake-domain.com
    # Istio Gateway LoadBalancer Domain
    service: http://192.168.2.85
  - service: http_status:404
```

Configuration file can be tested locally to validate if the tunnel works as expected.

```bash
cloudflared tunnel --config cloudflared.yaml run {ID del tunel}
```

To host the tunnel within Kubernetes, a Deployment must be created to run the CLI, in addition to a ConfigMap with file configuration and the secret with tunnel credentials.


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflare-config
  namespace: istio-system
data:
  config.yaml: |-
    tunnel: {tunnel ID}
    credentials-file: /etc/cloudflared/{tunnel ID}.json
    ingress:
      - hostname: fake-domain.com
        service: http://192.168.2.85
      - service: http_status:404
---
# The JSON with the credentials is saved by default in /home/{username}/.cloudflared/{tunnel ID}.json
# cat /home/{username}/.cloudflared/{ID del tunel}.json | base64 -w0
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-fake-domain-json
  namespace: istio-system
type: Opaque
data:
  {tunnel ID}.json: ****
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflare-fake-domain-pre
  namespace: istio-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudflare-fake-domain
  template:
    metadata:
      labels:
        app: cloudflare-fake-domain
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args: ["tunnel", "--no-autoupdate", "run", "{ID del túnel}"]
          volumeMounts:
            - name: configuration
              mountPath: /etc/cloudflared
              readOnly: true
      volumes:
        - name: configuration
          projected:
            sources:
              - configMap:
                  name: cloudflare-fake-domain-config
                  items:
                    - key: config.yaml
                      path: config.yaml
              - secret:
                  name: cloudflare-fake-domain-json
                  items:
                    - key: {tunnel ID}.json
                      path: {tunnel ID}.json
```

If everything is correct, redirecting the domain will expose the services; the services should be accessible over the internet; otherwise, there's a problem.

# Conclusions

Now, when someone accesses the website, the metric will be sent from the website to plausible via the domain plausible.fake-domain.com, passing through the Cloudflare tunnel, which in turn redirects the request to Istio, which in turn redirects 
the request to the service, applying the VirtualService policies.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/esquema-complejo-envio_en.png" >}}

For outbound traffic, the flow is very similar, but with one caveat: VirtualService policies only serve to filter inbound traffic. Istio allows you to control a service's outbound traffic in different ways. In this specific scenario, you would need 
to configure the Plausible namespace sidecar and use a ServiceEntry to define a whitelist of allowed external services and, optionally, an Egress Gateway to centralize and audit all outbound communication. The sidecar would be responsible for enforcing
these policies, blocking any unauthorized connections.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/esquema-complejo-respuesta_en.png" >}}

After looking in detail at how service exposure can be implemented cost-effectively, the question remains: is it a useful solution for production environments? Don't get me wrong, for my pre-production environment, it's a viable solution. However, does it
live up to expectations?

#### Security

Istio is quite flexible when applying policies, but for the most robust implementation, it's necessary to audit traffic connecting Istio to Prometheus and ELK/Loki, in addition to enabling a backup system. Cloudflare and Istio must also be configured to 
prevent a DDOS attack, although the tunnel acts as an implicit limiter, as only a finite number of persistent connections are initiated.

#### Performance

Additional latency is introduced through network hops (Cloudflare -> Tunnel -> Istio). Additionally, bandwidth is limited by the capacity of the tunnel and the node where Cloudflare is running. While this is sufficient for most services, it may not be ideal 
for workloads requiring minimal latency or massive scalability. It would be necessary to verify that it meets expectations by running a performance test.

#### Availability

This architecture introduces external dependencies (Cloudflare) and increases internal complexity. A configuration problem in any of these layers can lead to a service outage. Keeping this stack updated and monitored requires specific knowledge and can be 
resource-intensive. For a productive environment, it is vital to have a backup plan and a good understanding of the entire stack. An agent could be created that, after an alarm is triggered at one of the points (Cloudflare becomes unavailable), deploys 
the necessary infrastructure to expose the services to another cloud service.

#### Usability

If you manage hundreds of services with multiple development teams, you'll need to do extra work to create the necessary automations to configure deployment using IaC. When managing the solution en masse, it's also necessary to apply common organizational
policies regarding security, observability, and automation, avoiding unnecessary developer complexity.
