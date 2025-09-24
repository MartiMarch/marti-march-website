+++
title = "Exponiendo servicios a internet con Cloudflare y Istio"
tags = ["k8s", "istio", "cloudflare", "network"]
date = "2025-09-12"
lang = "es"
+++


# Introducción

Siempre he tenido curiosidad por saber quién visita mis páginas web, algunas de ellas hospedadas en GitHub. Es por eso que instalé Plausible para conocer métricas SEO, cosa que implicó tener que exponer ciertos servicios de mi entorno
self-hosted. Pero, ¿cómo se puede exponer un servicio a internet sin tener que lidiar con DDNS lentos o la contratación de una IP pública? Eso es lo que detallaré en este artículo: con la combinación de los túneles de Cloudfalred y Istio
para exponer los servicios.

# Dominios y DNS

Lo único que se necesita pagar son los dominios. Se pueden contratar en diversos proveedores, en mi caso he optado por emplear el servicio que tiene el propio Cloudfared. En general, los dominios terminados en *.com suelen rondar ~10
euros al año.

Una vez que tengas contratado el dominio, es necesario entender que es el CNAME, ya que afectará al redireccionar el tráfico. El CNAME es un tipo de registro DNS que indica que un dominio es un alias de otro dominio; cuando alguien escribe tu dominio 
en el navegador, el DNS lo redirige al dominio “real” que tiene los recursos. ¿Por qué es importante esto? Como Istio va a emplearse como Reverse Proxy y necesitamos exponer tantos servicios como sea posible, Cloudflare necesita saber a quién
ha de redireccionar la petición. Pero, si no se va a contratar una IP estática y pública, ¿cómo puede conocer el DNS a quién tiene que redirigir la petición? El que resuelve dicho problema es el túnel de Cloudflare. El túnel es un
deplyoment que, dentro del contenedor de este, se abre un socket con cierta configuración reapuntando a Istio. En otras palabras, Cloudflare funciona como un reverse proxy externo, que a su vez reenvía las peticiones a Istio, otro reverse proxy 
que distribuye el tráfico a los servicios dentro del clúster.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/tunel.png" >}}

Para que el dominio pueda ser redireccionado por el túnel, el CNAME tiene que usar un wildcard; de esta forma Istio puede redirigir a infinitos servicios usando un DNS wildcard como {{< inlines/black-inline "*.fake-domain.com" >}}. En mi caso se trata
de {{< inlines/black-inline "pluasible.fake-domain.com" >}}. Se puede configurar en el DNS del dominio desde la UI de Cloudflare.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/cname.png" >}}

# Istio

Instalar Istio en Kubernetes es sencillo utilizando el chart de Helm, no he necesitado modificar nada.

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
helm install istiod istio/istiod -n istio-system --wait
helm install istio-ingress istio/gateway -n istio-system
```

Necesitaremos dos piezas para poder redireccionar peticiones desde el deployment del túnel de Cloudflare a los servicios internos del clúster: un Gateway y un VirtualService. Un Gateway actúa como el punto de entrada del tráfico al
clúster, configurando los puertos, protocolos y certificados TLS que acepta el load balancer. Un VirtualService es el semáforo que define las reglas de enrutamiento para dirigir o bloquear las solicitudes entrantes (del Gateway)
hacia el servicio interno.

El Gateway y el VirtualService se han configurado para exponer de forma segura los endpoints de la API de Plausible, permitiendo únicamente el envío de métricas desde el script instalado en la página web my-fake-webpage.com. El VirtualService
solo permite el acceso a las dos rutas necesarias de Plausible para recibir datos (/js y /api/event), redirigiendo las peticiones entrantes al servicio interno del clúster. También se ha implementado una política para que solo la URL de la web
pueda enviar las peticiones, bloqueando cualquier otro acceso malintencionado al panel de administración de Plausible y redirigiendo el tráfico no deseado a un dominio falso.

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
        # Con el wildcard se puede redireccionar el wildcard del CNAME del dominio
        - "*.fake-domain.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  # Yo voy a redirigir todo mi tráfico a Plausible
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
            # Plausible envía métricas al path /js de plausible, solo se expone ese path
            # en caso contrarío cualquiera podría intentar hacer log-in en el servicio
            prefix: "/js"
      route:
        - destination:
            # DNS del servicio de Kubernetes, apunta aun ClusterIP
            host: plausible-http.infra.svc.cluster.local
            port:
              number: 8000
      corsPolicy:
        allowOrigins:
          # Solo acepto recibir peticiones desde el dominio de mi página web con el 
          # script de Plausible configurado
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
            # Este es otro endpoint que hay que exponer para el envío de métricas
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
    # Cae en un fallo en caso de solicitar otro path
    - route:
        - destination:
            host: blackhole.svc.cluster.local
```


Puedes validar si se está redireccionando la petición correctamente con el siguiente curl:

```bash
curl -v -H "Host: fake-domain.com" http://192.168.2.85/{service path}
```

# Cloudflare tunnel

Con el CLI de Cloudflare se puede levantar un túnel usando un par de comandos, exponiendo cualquier servicio local al exterior con un dominio generado aleatoriamente.

```bash
# Instalación del CLI
brew install cloudflared

cloudflared tunnel --url http://localhsot:8000
```

Para redireccionar al domino que se ha comprado, hay que realizar un inicio de sesión en Cloudflare para obtener el json con las credenciales. También hay que configurar la redirección del DNS.

```bash
# El comando abrirá una ventana donde se ha de seleccionar el dominio comprado
cloudflared tunnel login

# Permite que el DNS redireccione a Istio
cloudflared tunnel route dns {ID del tunel} fake-domain.com
```

Hay que crear un archivo de configuración para que el túnel conozca el dominio/IP al cual redireccionar, en este caso es la IP del Gateway de Istio.

```yaml
tunnel: {ID del tunel}
# Credenciales del túnel
credentials-file: /home/{username}/.cloudflared/{ID del tunel}.json

ingress:
  - hostname: fake-domain.com
    # Dominio del LoadBalancer del gateway de Istio
    service: http://192.168.2.85
  - service: http_status:404
```

Este archivo de configuración se puede usar localmente para probar si el túnel funciona como debería.

```bash
cloudflared tunnel --config cloudflared.yaml run {ID del tunel}
```

Para hospedar el túnel en Kubernetes hay que crear un deployment para el CLI, un configmap para la configuración y un secreto con las credenciales del túnel.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflare-config
  namespace: istio-system
data:
  config.yaml: |-
    tunnel: {ID del túnel}
    credentials-file: /etc/cloudflared/{ID del túnel}.json
    ingress:
      - hostname: fake-domain.com
        service: http://192.168.2.85
      - service: http_status:404
---
# El JSON con las credenciales se guarda por defecto en /home/{username}/.cloudflared/{ID del tunel}.json
# cat /home/{username}/.cloudflared/{ID del tunel}.json | base64 -w0
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-fake-domain-json
  namespace: istio-system
type: Opaque
data:
  {ID del tunel}.json: ****
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
                    - key: {ID del túnel}.json
                      path: {ID del túnel}.json
```

Si todo está bien redireccionado, el dominio ya expondrá los servicios. Los servicios tendrían que ser accesibles a través de internet, en caso contrario significa que hay algún fallo.

# Conclusiones

Ahora, cuando alguien acceda a la página web, se enviará la métrica desde la página web a plausible a través del dominio plausible.fake-domain.com, pasando por el túnel de cloudflare, que a su vez redirecciona la petición a Istio y, este
a su vez redirecciona la petición al servicio aplicando las políticas del VirtualService.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/esquema-complejo-envio.png" >}}

Para el tráfico de salida el flujo es muy similar, pero con un matiz: las políticas del VirtualService solo sirven para filtrar el tráfico entrante. Istio permite controlar el tráfico saliente de un servicio de diferentes formas.
En este escenario concreto se tendría que configurar el sidecar del namespace de Plausible y utilizar un ServiceEntry para definir una lista blanca de servicios externos permitidos y, opcionalmente, un Egress Gateway para centralizar y 
auditar toda la comunicación saliente. El sidecar sería el encargado de hacer cumplir estas políticas, bloqueando cualquier conexión no autorizada.

{{< images/centered-image src="/posts/expose-service-to-internet-with-cloudflare/esquema-complejo-respuesta.png" >}}

Tras ver en detalle como se puede implementar de forma económica la exposición de los servicios, queda preguntarse si es una solución útil para entornos productivos. Para mi entorno preproductivo es una solución solvente.
No obstante, ¿cumple las expectativas?

#### Seguridad

Istio es bastante flexible al aplicar políticas, pero para tener la mejor implementación robusta es necesario auditar el tráfico conectando Istio a Prometheus y ELK/Loki, además de habilitar un sistema de backup. También hay que configurar 
Cloudflare y Istio para evitar un ataque DDOS, aunque el túnel actúa como un limitador implícito, ya que solo se inicia un número finito de conexiones persistentes.

#### Rendimiento

Se introduce latencia adicional por los saltos de red (Cloudflare -> Túnel -> Istio). Además, el ancho de banda está limitado por la capacidad del túnel y del nodo donde se ejecuta Cloudflare. Si bien es suficiente para la mayoría de 
servicios, puede no ser ideal para cargas de trabajo que requieran latencia mínima o una escalabilidad masiva. Sería necesario verificar que cumple las expectativas lanzando una prueba de rendimiento.

#### Disponibilidad

Esta arquitectura introduce dependencias externas (Cloudflare) y aumenta la complejidad interna. Un problema en la configuración de cualquiera de estas capas puede derivar en una caída del servicio. Mantener este stack actualizado y 
monitorizado requiere conocimientos específicos y puede llegar a consumir demasiados recursos. Para un entorno productivo, es vital tener un plan de respaldo y un buen entendimiento de toda la pila. Se podría crear un agente que, tras
saltar una alarma en alguno de los puntos (por ejemplo, si Cloudflare deja de estar disponible) despliegue la infraestructura necesaria para exponer los servicios con otro servicio cloud.

#### Escalabilidad

Si se manejan cientos de servicios, con varios equipos de desarrollo, es necesario hacer un trabajo extra par crear los automatismos necesarios que permitan configurar el despliegue mediante IaC. Al gestionar masivamente la solución,
también es necesario aplicar unas políticas comunes a nivel organizativo a nivel de seguridad, observabilidad y automatización, evitando liar al desarrollador de forma innecesaria.
