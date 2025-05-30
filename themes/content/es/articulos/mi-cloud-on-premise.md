+++
title = "Cómo construí mi cloud privado sobre infraestructura on-premise"
tags = ["k8s", "sysadmin", "linux", "cloud", "devops", "on-premise"]
date = "2025-05-06"
lang = "es"
+++

# Introducción

En este artículo voy a tratar de explciar cómo y porqué decidí crear un cloud privado sobre una infraestructura on-premise, usando ordenadores lo más baratos posibles.

Si también tienes en mente emplear un cloud privado on-premise, has de saber que tiene cierto grado de complejidad y se requieren ciertos conocimientos previos de red, Kubernetes y Linux.

- [Capítulo 1: El problema](#capítulo-1-el-problema)
- [Capítulo 2: La solución](#capítulo-2-la-solución)
  - [Red energética](#red-energética)
  - [Arquitectura de red](#arquitectura-de-red)
  - [Despliegue del cluster de k8s](#desplieuge-del-cluster-de-k8s)
  - [Persistencia de la información](#persistencia-de-la-información)
- [Capítulo 3: La solución se transforma en un problema](#capítulo-3-la-solución-se-transforma-en-un-problema)
- [Capítulo 4: Colcusiones](#capítulo-4-conclusiones)

# Capítulo 1: El problema

Lo que más me gusta de trabajar en informática es que tengo la posibilidad de construir muchas cosas con un ordenador. La idea de generar ingresos extra siempre ronda por mi cabeza. Pero solo soy una persona, no una compañía. Necesito optimizar
mis recursos estratégicamente o podría llegar a derrochar mucho tiempo y dinero. Además, si ya de por sí un proyecto puede encarecerse, en el mundo tecnológico los costes pueden dispararse sin control. También es importante ser precavido con ciertas cuestiones
legales relacionadas con la titularidad y las condiciones de uso, es decir, con la propiedad intelectual. Compartir código abiertamente, sin establecer ninguna restricción, puede llegar a ser una mala idea. Soy consciente de que, muy probablemente, 
jamás logre monetizar ninguna de mis soluciones... pero, ¿quién sabe?

Por todo ello llegué a una conclusión: para evitar problemas, debía contar con mi propia infraestructura. La estrategia es simple: por un lado, evito tener que pagar por servicios en la nube o soluciones SaaS y, por otro, mantengo todos mis proyectos
ocultos al ojo ajeno. Me incomoda pensar que los costes puedan descontrolarse por un simple error de configuración en una infraestructura de terceros. Además, me ahorro tener que crear una sociedad limitada solo para proteger legalmente lo que hago. 
Una SL conlleva demasiados quebraderos de cabeza a nivel burocrático en el país donde vivo, España.

# Capítulo 2: La solución

Para crear algo mínimanete funcional primero es necesario establecer el paradigma sobre el que ir tomando las decisiones. En mi caso he establecido tres principios:
- Las soluciones creadas se han de poder desplegar en otros entornos cloud, como podrían ser AWS, Azure, GCP o directamente sobre un cluster de Kubernetes.
- Se ha de probar la solución sobre un entorno distribuido y con alta disponiblididad (HA).
- Cada compontente que conforma el sistema had e ser lo más barato posible

La tecnología trabaja en base a capas de abstarcción, es responsabilidad de cada uno decidir a partir de que nivel  empezar a asumir responsabilidades o, por el contrario, cederlas a terceros. En este caso contreto decidí partir con máquinas físicas
sobre las que instalé Ubuntu para terminar desplegando un cluster de Kubernetes. Considerando que cada nodo requiere 2 cores y 2 Gb como mínimo para poder operar como nodo del cluster, compré seis máquinas. Tres de estas tiene el rol de Control Plane y
las tres restantes el rol de Worker. En caso de necesitar más capacidad se pueden incrementer el número de nodos Worker, así que hay bastante flexibilidad.

Lo que hice a continuacióin fue seleccionar los componentes de ordenador que mejor se ajustaban, es decir, los más baratos.

| Role     | CPU          | RAM   | Motherboard           | PSU   | Disk               |
|----------|--------------|-------|-----------------------|-------|--------------------|
| master 1 | Ryzen 3 4100 | 12 GB | GigaByte A520M S2H    | 550 W | 120GB SSD          |
| master 2 | Ryzen 3 4100 | 12 GB | GigaByte A520M S2H    | 550 W | 120GB SSD          |
| master 3 | Ryzen 3 4100 | 12 GB | ASRock A320M-DVS R4.0 | 550 W | 120GB SSD          |
| worker 1 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |
| worker 2 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |
| worker 3 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |

Algunas métricas contables de lo que cuestan las piezas y poner a funcionar ciertas confioguraciones de cluster:

#### Componentes
- Ryzen 3 4100: {{< mathematics/inline >}}~51,80€{{< /mathematics/inline >}}
- Ryzen 3 4100: {{< mathematics/inline >}}~155€{{< /mathematics/inline >}}
- 4GB RAM: {{< mathematics/inline >}}21€{{< /mathematics/inline >}}
- 8GB RAM: {{< mathematics/inline >}}~18,47€{{< /mathematics/inline >}}
- GigaByte A520M S2H: {{< mathematics/inline >}}58,95€{{< /mathematics/inline >}}
- ASRock A320M-DVS R4.0: {{< mathematics/inline >}}62,51€{{< /mathematics/inline >}}
- PSU (550W): {{< mathematics/inline >}}~29€{{< /mathematics/inline >}}
- 120GB SSD:{{< mathematics/inline >}} 14€{{< /mathematics/inline >}}
- 1TB SSD: {{< mathematics/inline >}}58,99€{{< /mathematics/inline >}}

#### Nodos
- Control plane cost (master*): {{< mathematics/inline >}}193,22€{{< /mathematics/inline >}}
- Worker node (worker*): {{< mathematics/inline >}}374,91€{{< /mathematics/inline >}}

#### Diferentes escenarios de clusters
- Single node cluster: {{< mathematics/inline >}}374,91€{{< /mathematics/inline >}}
- Minimal cluster (1 control plane and 1 worker node): {{< mathematics/inline >}} 568,13€ {{< /mathematics/inline >}}
- Worker HA cluster (1 control plane and 2 worker nodes): {{< mathematics/inline >}} 943,04€ {{< /mathematics/inline >}}
- Minimal HA cluster (2 control planes and 2 worker nodes): {{< mathematics/inline >}} 1136,26€ {{< /mathematics/inline >}}
- **Current HA cluster (3 control planes and 3 worker nodes): {{< mathematics/inline >}} 1704,39€ {{< /mathematics/inline >}}**

Lo que tenemos hasta ahora no son nada más que máquinas, para que se transformen en parte de un cluster de Kubernetes  hay que controlar tres pilaes: la energia, la persistencia de los datos y la configuración de la red. Por supuesto, hay muchas otras
cosas, como por ejemplo el sistema de alarmado, una estrategia de disaster recovery, la seguridad... No obstante, si nos ceñimos al objetivo de crear un cluster "minimamente" funcional, hay que manejar los tres aspectos anteriores.

### Red energética

Busqué los planos de la red elećtrica de mi casa para revisar la energía máxima que entrega cada línea. La idea es confirmar que, al conecatr tantos ordenadores, el diferencial no salte por superar la carga máxima. Por eso hay que calcular primero
cuantos dispositivos soporta cada línea con la siguiente fórmula:

{{< mathematics/block >}}
$$
amperios = \frac{watts}{voltios} \rightarrow A = \frac{W}{V} \rightarrow A = \frac{W}{240V \text{(por defecto en España)}}
\\
A_{\text{max psu}} = \frac{550W}{240V} = 2.29A
$$
{{< /mathematics/block >}}

Un nodo funcionando a máxima potencia (consumo de Watts), supone en total unos {{< mathematics/inline >}}2.29A{{< /mathematics/inline >}}. Pero cada fuente de alimentación no va a demandar el 100% de energia posible ya que los componentes que vamos
a conectar no llegan a dicho límite, por eso hay que ajustar por nodo el consumo máximo real ([página utilizada](https://www.geeknetic.es/calculadora-fuente-alimentacion/) para obtener los watts de cada componente).

| Rol      | Watts máximos | Amperios máximos | Line de energía |
|----------|---------------|------------------|-----------------|
| master 1 | 154 W         | 0.64 A           | Linea 1         |
| master 2 | 154 W         | 0.64 A           | Linea 2         |
| master 3 | 154 W         | 0.64 A           | Linea 2         |
| worker 1 | 157 W         | 0.65 A           | Linea 1         |
| worker 2 | 157 W         | 0.65 A           | Linea 1         |
| worker 3 | 157 W         | 0.65 A           | Linea 2         |
| Router   | ~20 W         | 0.08 A           | Linea 1         |
| Switch   | ~20 W         | 0.08 A           | Linea 1         |

Cada línea ofrece unos 16 amperios, el consumo final de cada nodo solicitando la máxima energía teórica no supera lo esperado.

{{< mathematics/block >}}
$$
A_{\text{linea 1}} = \frac{ W_{\text {master 1}} + W_{\text {worker1}} + W_{\text {worker 2} } + W_{\text {Router}} + W_{\text {Switch}}}{240V} \approx 2,11A
\\
\text{Uso de la Linea 1} = \frac{ 2,11A }{ 16A } * 100 \approx 13,18\%
\\
A_{\text {linea 2}} = \frac{ W_{\text {master2}} + W_{\text {master3}} + W_{\text {worker3}} }{240V} \approx 1,93A
\\
\text{Uso de la Linea 2} = \frac{ 1,93A }{ 16A } * 100 \approx 12,06\%
$$
{{< /mathematics/block >}}

Gracias a todos estos cálculos pude confirmar que no se producirá ningún apagón en mi casa.

### Arquitectura de red

Hay muchas maneras de diseñar la topología de una red. En esta caso decidí aislar el entorno donde se aloja el cluster de la red doméstica, para evitar interferencias. Para ello usé un router intermedio, creando asñí el conocido WAN-to-LAN: la subred
192.168.1.0/24 se mantiene como red doméstica minetras que la subred 192.168.2.0/24 es donde se aloja el cluster. Para mantener una coherencia en la asignación de IP's asocié un rango de IP's con ciertos roles:

- {{< mathematics/inline >}}192.168.2.0{{< /mathematics/inline >}}: Unassigned by now (_it would be another router to enable HA_)
- {{< mathematics/inline >}}192.168.2.1{{< /mathematics/inline >}}: IP router
- {{< mathematics/inline >}}192.168.2.2 - 192.168.2.49{{< /mathematics/inline >}}: DHCP
- {{< mathematics/inline >}}192.168.2.50 - 192.168.2.59{{< /mathematics/inline >}}: Nodos control plane del cluster de Kubernetes
- {{< mathematics/inline >}}192.168.2.60{{< /mathematics/inline >}}: IP virtual aisgnada al control plan con el rol de líder
- {{< mathematics/inline >}}192.168.2.61 - 192.168.2.89{{< /mathematics/inline >}}: Nodos worker del cluster de Kubernetes
- {{< mathematics/inline >}}192.168.2.90 - 192.168.2.99{{< /mathematics/inline >}}: Reservado para los sistemas de alamcenamiento de datos (_e.g. NAS_)
- {{< mathematics/inline >}}192.168.2.100 - 192.168.2.255{{< /mathematics/inline >}}: Sin asignar

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/network-topology-1.png" >}}

Para crear un cluster de Kubernetes en HA no basta tan solo con tener varios nodos worker, también se necesita contar con varios control plane, asociando estos ultimos a una IP virtual. El funcionamiento es simple, si un nodo de tipo control plane
se cae, otro tomará su lugar asumiendo el rol de líder (algoritmo RAFT). Para hacer funcionar algo así combiné dos herramientas amplaimente usadas, HaProxy y KeepAlived.

Lo que ocurre por debajo es la siguiente secuencia:

1. KeepAlived asigna el rol de líder a algún control plane disponible

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_1.png" >}}

2. La IP virtual se asocia con el control plane líder del punto anterior. A partir de este momento nodo en custión es elñ encargado de comunicarse con los nodos worker.

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_2.png" >}}

3. El control plane líder deja de dar servicio

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_3.png" >}}

4. Los control plane que siguen funcionando empiezan a negociar quien será el nuevo líder. Una vez establecido, KeepAlived asociará la IP virtual al nuevo nodo líder.

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_4.png" >}}

Tanto HaProxy como KeepAlived se han de instalar en cada control plane. Los pasos para su instalación son los mismos, lo único que difire son los archivos de configuración:

```bash
apt -y update
apt -y install vim
apt -y install keepalived
apt -y install heproxy
```

Contenido de /etc/haproxy/haproxy.cfg file:

```plaintext
global
log /dev/log  local0 warning
chroot      /var/lib/haproxy
pidfile     /var/run/haproxy.pid
maxconn     4000
user        haproxy
group       haproxy
daemon

stats socket /var/lib/haproxy/stats

defaults
  log global
  option  httplog
  option  dontlognull
  timeout connect 5000
  timeout client 50000
  timeout server 50000

frontend kube-apiserver
  # Se trata del puerto al que atacarán todos los nodos worker
  # No se puede configurar el puerto 6443 ya que entraría en conflicto con el servicio de kubeadm
  bind *:7443
  mode tcp
  option tcplog
  default_backend kube-apiserver

backend kube-apiserver
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  # Lista de nodos control plane
  server kube-apiserver-1 192.168.2.51:6443 check
  server kube-apiserver-2 192.168.2.52:6443 check
  server kube-apiserver-3 192.168.2.53:6443 check
```

Contenido de /etc/keepalived/keepalived.conf

```plaintext
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}

vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}

vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface enp4s0
  virtual_router_id 60
  advert_int 1
  authentication {
  auth_type PASS
    auth_pass 1111
  }
  # IP del control plane que se esta configurando
  unicast_src_ip 192.168.2.51
  unicast_peer {
    # IP's de los otros control plane
    192.168.2.52
    192.168.2.53
  }

  virtual_ipaddress {
    # IP virtual con la que se comunicarán los worker
    192.168.2.60/24
  }

  track_script {
    chk_haproxy
  }
}
```

Como último paso hay que habilitar los servicios y arrancarlos:

````bash
systemctl enable haproxy
systemctl enable keepalived
systemctl start haproxy
systemctl start keepalived
````

### Desplieuge del cluster de K8S

Para desplegar Kubernetes utilicé una herramienta llamada KubeKey. Ha sido desarrollada por la comunidad KubeSphere, se trata de un CLI que simplifica la isntalación conectandose por SSH a cada nodo.

```bash
curl -sfL https://get-kk.kubesphere.io | VERSION={versión} sh -
```

Antes de realizar la instalación hay que configurar un Yaml con todas las IP de los nodos y sus respectivos roles.

```yaml
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  # Todos los nodos que conformarán el clsuter
  - {name: master1, address: 192.168.2.51, internalAddress: 192.168.2.51, user: master1, password: ""}
  - {name: master2, address: 192.168.2.52, internalAddress: 192.168.2.52, user: master2, password: ""}
  - {name: master3, address: 192.168.2.53, internalAddress: 192.168.2.53, user: master3, password: ""}
  - {name: worker1, address: 192.168.2.61, internalAddress: 192.168.2.61, user: worker1, password: ""}
  - {name: worker2, address: 192.168.2.62, internalAddress: 192.168.2.62, user: worker2, password: ""}
  - {name: worker3, address: 192.168.2.63, internalAddress: 192.168.2.63, user: worker3, password: ""}
  roleGroups:
    etcd:
    # Alias usados para los nodos control plane
    - master1
    - master2
    - master3
    control-plane:
    - master1
    - master2
    - master3
    worker:
    # Alias usados para los nodos worker
    - worker1
    - worker2
    - worker3
  controlPlaneEndpoint:
    domain: cluster.noprod.local
    # IP virtual configurada previamente con KeepAlived y HaProxy
    address: "192.168.2.60"
    # Puerto definido en HaProxi
    port: 7443
  kubernetes:
    version: v1.23.10
    clusterName: cluster.noprod
    autoRenewCerts: true
    # Orquestador de contenedores, se puede utilizar otro
    containerManager: docker
  etcd:
    type: kubekey
  network:
    # Plugin de red de Kubernetes, se puede utilizar otro
    plugin: flannel
    # Red que utilizará el cluster para desplegar los contenedores. No puede entrar en conflicto con otras subredes (en mi caso 192.168.2.0/24).
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    multusCNI:
      enabled: false
  registry:

    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons: []
```

Para desplegar el clsuter tan solo hay que ejecutar este último comando:

```bash
chmod +x kk
./kk create cluster -f {kube key cluster configuration}.yaml
```

### Persistencia de la información


1. Se deshabilita el mutlipath en cada nodo worker

```bash
vi /etc/multipath.conf
```

```plaintext
blacklist {
    devnode "^sd[a-z0-9]+"
}
```

2. Instalación de Longhorn

La persistencia de la información es uno de los aspectos más desafiantes en los sistemas distribuidos. He probado muchas soluciones, y la que elegí fue mantener los datos replicados. En este enfoque, los datos se tratan como si fueran un pod:
configuras el número deseado de réplicas y el sistema se encarga de que los datos permanezcan replicados en los discos de los nodos worker. Si un nodo se cae, un volumen persistente volverá a adjuntar los datos desde otro nodo disponible
mientras se clona la copia de datos que sigue estando sana. Por eso se ha añadido un SSD adicional de 1 TB a cada nodo worker. Todo esto se logra utilizando Longhorn, aunque hay otras herramientas que permiten implementar la misma estrategia.

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

3. Configuración de disco de los nodos worker

```bash
lsblk
fdisk /dev/vdb # n, p, 1, w
mkfs.ext4 /dev/<SSD disk name>
mkdir /mnt/ssd-1 # Puede configurarse otro nombre
mount /dev/vdb1 /mnt/ssd-1
vi /etc/fstab # Hay que agregar la siguiente línea en /etc/fstab -> "/dev/vdb1 /mnt/ssd-1 ext4 defaults 0 0"
df -h
```

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/longhorn_1.png" >}}

4. Definición del manifiesto del PVC

Cada vez que se quiera replicar el dato en diferentes nodos hay que usar el storage class de Longhorn apuntando a un volumen. El volumen se puede crear tanto desde la UI como automáticamente cuando el pv solicita el recurso.

```yaml
# Manifiesto de ejemplo de un PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitea-ha-pv-data
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 20Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: gitea-ha-pvc-data
    namespace: infra
  csi:
    driver: driver.longhorn.io
    fsType: ext4
    volumeHandle: gitea-ha
  storageClassName: longhorn
  volumeMode: Filesystem
```

```yaml
# Manifiesto de ejemplo de un PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-ha-pvc-data
  namespace: infra
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: longhorn
  volumeMode: Filesystem
  volumeName: gitea-ha-pv-data
```

### Exposición de servicios

Hasta ahora tenemos montado un clúster de Kubernetes con alta disponibilidad, interesante, ¿Verdad? Sin embargo, sin exponer los servicios al exterior, solo es un monton de máquinas sin mucha utilidad.

Existen dos formas pde hacer accesibles los servicios fuera del clúster: mediante NodePort o LoadBalancer. Usar NodePort no tiene mucho sentido en un entorno HA, ya que el servicio queda vinculado a un único nodo, lo que introduce un posible punto
único de fallo. Una alternativa sería montar un balanceador de carga con HAProxy y KeepAlived, usando un nuevo grupo de máquinas. Aunque esto funcionaría, también implica un mayor coste, ya que necesitarías al menos dos nuevos ordenadores. Una opción
más barata consiste en utilizar directamente los propios nodos worker como balanceadores de carga, aprovechando OpenELB.

Al igual que antes, hay muchas herramientas disponibles. En mi caso, he seleccionado [OpenELB en modo Layer 2](https://openelb.io/docs/concepts/layer-2-mode/) para asignar una IP libre de mi subred 192.168.2.0/24 al Ingress. A grandes rasgos, así es
cómo funciona:
- OpenELB asigna una IP virtual perteneciente a la subred real (no a la red superpuesta de Kubernetes) al servicio.
- El router añade esa IP a su tabla ARP.
- La IP virtual se vincula a uno de los nodos worker. Si ese nodo se cae, OpenELB reasigna la IP a otro nodo worker que esté operativo.

Para instalar OpenELB se pueden usar los manifiestos que hay disponibles en la documentación.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
kubectl apply -f https://raw.githubusercontent.com/openelb/openelb/master/deploy/openelb.yaml
kubectl -n openelb-system get pod
```

La subred usada por OpenELB, al asignar las IP virtuales, tiene que especificarse mediante el CRD Eip:

```yaml
apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
  name: layer2-eip
spec:
  # Subred a la que se atará el ingress
  address: 192.168.2.80-192.168.2.200 
  # Interfaz de red física de los nodos worker
  interface: enp6s0
  protocol: layer2
```

Cada vez que se despliegue un servicio de tipo LoadBalances, OpenELB le asignará una IP de la subred previamente configurada.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: layer2-svc
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    # Anotaciones necesarias si queremos que OpenELB se encargue de asignar la IP externa
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: layer2-eip
spec:
  selector:
    app: layer2-openelb
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 8080
  externalTrafficPolicy: Cluster
```

En este caso concreto, se ha asignado la IP 192.168.2.80 como dirección externa porque es la primera dentro del rango de direcciones EIP.

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/openelb_1.png" >}}

El último paso para tener una solución sólida consiste en asegurar el acceso al Ingress mediante una autoridad certificadora (CA). Se puede hacer con cert-manager, un servicio integrado en Kubernetes que gestiona automáticamente los certificados. 
Primero se tiene que crear una clave con la que firmar los certificados usando la CA.

```yaml
# Para crear la clave primaria
openssl genrsa -out ca.key 4096

# Para firmar cualquier certificado con la clave primaria de la CA
openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt
```

cert-manager almacena el certificado de la CA a través del CRD ClusterIssuer.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  ca:
    secretName: ca-cert-manager
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ca-cert-manager
  namespace: cert-manager
type: Opaque
data:
  tls.crt: # Contenido en Base64-encoded del ca.crt (e.g., cat ca.crt | base64 -w 0)
  tls.key: # Contenido en Base64-encoded del ca.key (e.g., cat ca.key | base64 -w 0)
```

Para exponer los servicio fuera del cluster, hay que combinar Nginx y cert-manager al definir el manifiesto del Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # Con esta anotación se le indica a cert-manager que utilice el ClusterIssuer especificado
    cert-manager.io/cluster-issuer: ca-cluster-issuer
  name: gitea-ha-gitea-ha
  namespace: infra
spec:
  # Especifica que este Ingress debe ser gestionado por el controlador de Ingress de NGINX.
  ingressClassName: nginx
  rules:
  - host: pre.gitea.com
    http:
      paths:
      - backend:
          service:
            name: gitea-ha-gitea-ha
            port:
              number: 3000
        path: /
        pathType: Prefix
  tls:
  - hosts:
    # cert-manager almacenará el certificado TLS emitido en este secreto.
    # Si no existe, cert-manager lo creará y lo renovará automáticamente.
    - pre.gitea.com
    secretName: gitea-ha-gitea-ha-tls
```

# Capítulo 3: La solución se transforma en un problema

Siguiendo esta guía, casi da la impresión de que cualquiera puede montar un clúster de Kubernetes con alta disponibilidad comprando unas cuantas tostadoras. Suena fácil, ¿verdad? Pero ya lo advertí al principio de este artículo: crear y mantener
un clúster de Kubernetes no es nada sencillo, especialmente cuando se ejecuta sobre infraestructura on-premise, con los recursos justos. He tenido que lidiar con muchos problemas solo para mantener el entorno funcionando, la mayoría como consecuencia
de mis malas decisiones.

En un momento dado, los nodos worker empezaron a reiniciarse aleatoriamente con un error de segmentación de memoria del kernel. Pasé mucho tiempo intentando averiguar la causa del problema. Tras varias semanas sin encontrar el origen, tiré la toalla.
Decidí copiar los datos de los volúmenes persistentes a mi ordenador personal y reinstalar todo el clúster desde cero. Pero... el error seguía apareciendo. Entonces empecé a repasar todos los cambios que había hecho recientemente, y me di cuenta del
verdadero problema: una ranura de RAM que había añadido no hacía mucho. Estaba funcionando a una frecuencia distinta al resto. Sí, me pasé horas investigand... y la solución era tan simple como reapsar todas las modificaciones que habia realizado.

Otro problema fue usar una conexión Molex a SATA. Al principio, antes de utilizar Longhorn, todos los datos se almacenaban en un host independiente que actuaba como almacenamiento de Kubernetes, usando NFS. La placa base de ese host solo tenía
seis puertos SATA, así que utilicé un adaptador Molex a SATA para aumentar la capaacidad de los discos y poder montar así un RAID. Con el tiempo, uno de los discos falló. Estoy bastante seguro de que fue por la alimentación inestable que proporcionaba 
la conexión Molex. En su momento parecía una solución rápida y efectiva, no obstante, acabó comprometiendo la estabilidad de toda la infraestructura y de los datos.


# Chapter 4: Conclusions

Como ingeniero, debes evaluar con cuidado si montar un clúster de Kubernetes on-premise realmente es la mejor opción. La lista de problemas que pueden surgir es interminable, enfrentarte a ellos consume muchísimo tiempo y energía. Aun así, si decides
construir uno, el conocimiento que obtendrás será muy valioso. Las lecciones que se aprenden al enfrentarse al reto de gestionar tu propio clúster son insustituibles y sin duda influirán positivamente en tu forma de tomar decisiones en futuros 
proyectos.
