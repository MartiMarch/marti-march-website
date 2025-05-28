+++
title = "How my on premise private cloud was built"
tags = ["k8s", "sysadmin", "linux", "cloud", "devops", "on-premise"]
date = "2025-05-06"
lang = "en"
+++

# Preface

In this post, I'm going to try to explain why and how I decided to build a private cloud on-premise using some cheap computers.

If you have in mind doing something like this, hosting your own infrastructure at home, let me confess: it means choose the hard way... Don’t say I didn’t warn you.

- [Chapter 1: The problem](#chapter-1-the-problem)
- [Chapter 2: The solution](#chapter-2-the-solution)
  - [Energy architecture](#energy-architecture)
  - [Network architecture](#network-architecture)
  - [Deploying K8S cluster](#deploying-k8s-cluster)
  - [Persistence of data](#persistence-of-data)
  - [Exposing services](#exposing-services)
- [Chapter 3: The solution becomes the problem](#chapter-3-the-solution-becomes-the-problem)
- [Chapter 4: Conclusions](#chapter-4-conclusions)

# Chapter 1: The problem

I enjoy creating new solutions and trying to make money. But I'm just one person, not a company. That means I need to optimize my spending, otherwise, I could end up wasting a lot of time and money. Trying to build something can get very expensive, and in 
the IT world, it can become massively expensive. So, here's my first problem: money (yeah, the usual one). The second problem is about intellectual property. When someone is working on a new solution, sharing it publicly without considering
the legal side can be risky—especially if there’s any hope of future profit. I know, I know... I'm aware I probably won't make a dime, but, who knows?

I came to the conclusion that, to avoid both problems, the infrastructure must be hosted on-premise. With this approach, I can skip the need of pay for an expensive cloud or any type of SaaS, keeping my projects hidden. On one hand, I don't need to
deal with a "dynamic" budget that could surprise me with nasty expenses. On the other one, I don't need to create a company. And that last point is key. I live in Spain, which means starting a company is expensive and involves fight with a lot of
bureaucratic headaches.


# Chapter 2: The solution

To achieve a functional system, a clear paradigm must be established first. What is the best option under my conditions? I've decided to follow three main rules:

- Developed applications must be easy to port to another environment, such as a cloud provider like AWS, Azure or GCP.
- The environment must allow me to test high availability (HA) and distributed solutions.
- Each component that comprises the environment must be as cost-effective as possible.

Technology operates with layers of abstraction, and one must decide which responsibilities are delegated to external systems and which are managed internally. That’s why I’ve chosen to work from the ground up—starting with physical machines
and building up to a Kubernetes cluster. Considering that a Kubernetes node requires at least 2 cores and 2 GB, I bought six machines. Three of them serve as control plane nodes and the other three as worker nodes. One of the benefits of using
Kubernetes is that the number and capacity of each node can vary on demand.

With this in mind, I chose the most coherent computer components (the cheapest ones):

| Role     | CPU          | RAM   | Motherboard           | PSU   | Disk               |
|----------|--------------|-------|-----------------------|-------|--------------------|
| master 1 | Ryzen 3 4100 | 12 GB | GigaByte A520M S2H    | 550 W | 120GB SSD          |
| master 2 | Ryzen 3 4100 | 12 GB | GigaByte A520M S2H    | 550 W | 120GB SSD          |
| master 3 | Ryzen 3 4100 | 12 GB | ASRock A320M-DVS R4.0 | 550 W | 120GB SSD          |
| worker 1 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |
| worker 2 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |
| worker 3 | Ryzen 5 4500 | 24 GB | ASRock A320M-DVS R4.0 | 550 W | 450GB SSD, 1TB SSD |

An overview of the cluster cost metrics:

#### Components
- Ryzen 3 4100: {{< mathematics/inline >}}~51,80€{{< /mathematics/inline >}}
- Ryzen 3 4100: {{< mathematics/inline >}}~155€{{< /mathematics/inline >}}
- 4GB RAM: {{< mathematics/inline >}}21€{{< /mathematics/inline >}}
- 8GB RAM: {{< mathematics/inline >}}~18,47€{{< /mathematics/inline >}}
- GigaByte A520M S2H: {{< mathematics/inline >}}58,95€{{< /mathematics/inline >}}
- ASRock A320M-DVS R4.0: {{< mathematics/inline >}}62,51€{{< /mathematics/inline >}}
- PSU (550W): {{< mathematics/inline >}}~29€{{< /mathematics/inline >}}
- 120GB SSD:{{< mathematics/inline >}} 14€{{< /mathematics/inline >}}
- 1TB SSD: {{< mathematics/inline >}}58,99€{{< /mathematics/inline >}}

#### Nodes
- Control plane cost (master*): {{< mathematics/inline >}}193,22€{{< /mathematics/inline >}}
- Worker node (worker*): {{< mathematics/inline >}}374,91€{{< /mathematics/inline >}}

#### Cluster cost sceneries
- Single node cluster: {{< mathematics/inline >}}374,91€{{< /mathematics/inline >}}
- Minimal cluster (1 control plane and 1 worker node): {{< mathematics/inline >}} 568,13€ {{< /mathematics/inline >}}
- Worker HA cluster (1 control plane and 2 worker nodes): {{< mathematics/inline >}} 943,04€ {{< /mathematics/inline >}}
- Minimal HA cluster (2 control planes and 2 worker nodes): {{< mathematics/inline >}} 1136,26€ {{< /mathematics/inline >}}
- **Current HA cluster (3 control planes and 3 worker nodes): {{< mathematics/inline >}} 1704,39€ {{< /mathematics/inline >}}**

To transform all this stuff into a real HA Kubernetes cluster, you have to manage three fundamentals: energy, data persistence and network configuration. Of course, there are many other things, like monitoring, disaster recovery, security...
However, if we take into consideration make it a "minimal" functional cluster, focus on these three things.

### Energy architecture

I reviewed the residential infrastructure dashboards to check the maximum power supply for each power line. The idea behind this is to determine how many devices can be connected to each power line without causing a power outage in the home
electrical system. To do that, I used the following formula:

{{< mathematics/block >}}
$$
amperes = \frac{watts}{volts} \rightarrow A = \frac{W}{V} \rightarrow A = \frac{W}{240V \text{(default in Spain)}}
\\
A_{\text{max psu}} = \frac{550W}{240V} = 2.29A
$$
{{< /mathematics/block >}}

A single node, performing at its maximum, gives a total of {{< mathematics/inline >}}2.29A{{< /mathematics/inline >}}. But the PSU only supplies the energy that is demanded. So, to calculate the actual values, we must adjust 
the formula according to the previous components at its maximum performance _([webpage used](https://www.geeknetic.es/calculadora-fuente-alimentacion/) to get component watts)_.

| Role     | Maximum watts | Maximum amperes | Power line |
|----------|---------------|-----------------|------------|
| master 1 | 154 W         | 0.64 A          | Line 1     |
| master 2 | 154 W         | 0.64 A          | Line 2     |
| master 3 | 154 W         | 0.64 A          | Line 2     |
| worker 1 | 157 W         | 0.65 A          | Line 1     |
| worker 2 | 157 W         | 0.65 A          | Line 1     |
| worker 3 | 157 W         | 0.65 A          | Line 2     |
| Router   | ~20 W         | 0.08 A          | Line 1     |
| Switch   | ~20 W         | 0.08 A          | Line 1     |


Each line supplies 16 amperes, so the final usage per line when computers are performing at {{< mathematics/inline >}}100%{{< /mathematics/inline >}} would be as follows:

{{< mathematics/block >}}
$$
A_{\text{line 1}} = \frac{ W_{\text {master 1}} + W_{\text {worker1}} + W_{\text {worker 2} } + W_{\text {Router}} + W_{\text {Switch}}}{240V} \approx 2,11A
\\
\text{Line 1 usage} = \frac{ 2,11A }{ 16A } * 100 \approx 13,18\%
\\
A_{\text {line 2}} = \frac{ W_{\text {master2}} + W_{\text {master3}} + W_{\text {worker3}} }{240V} \approx 1,93A
\\
\text{Line 2 usage} = \frac{ 1,93A }{ 16A } * 100 \approx 12,06\%
$$
{{< /mathematics/block >}}

With all this information, is it possible guarantee that a blackout won't happen.


### Network architecture

There are many ways to design network topologies. In this case, I decided to create a new isolated physical subnet to avoid interference between the cluster and domestic networks. By using an intermediate router, I created a WAN-to-LAN setup:
subnet 192.168.1.0/24 remains as the domestic network, and 192.168.2.0/24 as cluster subnet. To maintain a global IP coherency, nodes are configured according to their role:

- {{< mathematics/inline >}}192.168.2.0{{< /mathematics/inline >}}: Unassigned by now (_it would be another router to enable HA_)
- {{< mathematics/inline >}}192.168.2.1{{< /mathematics/inline >}}: IP router
- {{< mathematics/inline >}}192.168.2.2 - 192.168.2.49{{< /mathematics/inline >}}: DHCP range
- {{< mathematics/inline >}}192.168.2.50 - 192.168.2.59{{< /mathematics/inline >}}: Kubernetes control plane nodes
- {{< mathematics/inline >}}192.168.2.60{{< /mathematics/inline >}}: Virtual IP assigned to control plane HA
- {{< mathematics/inline >}}192.168.2.61 - 192.168.2.89{{< /mathematics/inline >}}: Kubernetes worker nodes
- {{< mathematics/inline >}}192.168.2.90 - 192.168.2.99{{< /mathematics/inline >}}: Reserved for persistence storage servers (_e.g. NAS_)
- {{< mathematics/inline >}}192.168.2.100 - 192.168.2.255{{< /mathematics/inline >}}: Unassigned


{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/network-topology-1.png" >}}

To build a HA kubernetes cluster, workers must communicate with a control plane, but, if it goes down, another must take over as the leader (_RAFT algorithm_).
To enable that, a virtual IP address is needed. It is commonly implemented combining HaProxy with KeepAlived.

It is what happens when a control node goes down:

1. KeepAlived assigns a leader role among the masters in the available pool

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_1.png" >}}

2. The virtual IP is associated with the master node in question and the worker nodes attack that control plane

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_2.png" >}}

3. A control plane goes down

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_3.png" >}}

4. KeepAlived negotiates the leader role again with the remaining master nodes and associates the virtual IP with it, the worker nodes attack that control plane

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/keepalive-haproxy_4.png" >}}

HaProxy and KeepAlived must be installed on each control plane. Installation steps are the same, the only difference is the configuration files of haproxy and keepalived.

```bash
apt -y update
apt -y install vim
apt -y install keepalived
apt -y install heproxy
```

Content of /etc/haproxy/haproxy.cfg file:

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
  # This port will be used by worker nodes to communicate 
  # Don't set 6443, as it would conflit with the control plane's kubeadm service
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
  # Poll of control planes that form the pool
  server kube-apiserver-1 192.168.2.51:6443 check
  server kube-apiserver-2 192.168.2.52:6443 check
  server kube-apiserver-3 192.168.2.53:6443 check
```

Content of /etc/keepalived/keepalived.conf

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
  # Control plane IP
  unicast_src_ip 192.168.2.51
  unicast_peer {
    # IP's of other control plane
    192.168.2.52
    192.168.2.53
  }

  virtual_ipaddress {
    # IP addres of virtual control plane
    192.168.2.60/24
  }

  track_script {
    chk_haproxy
  }
}
```

As last step, enable both services:

````bash
systemctl enable haproxy
systemctl enable keepalived
systemctl start haproxy
systemctl start keepalived
````

### Deploying K8S cluster

To deploy the Kubernetes cluster, I used [KubeKey](https://www.kubesphere.io/docs/v3.3/installing-on-linux/high-availability-configurations/set-up-ha-cluster-using-keepalived-haproxy/), a tool developed by the KubeSphere community that simplifies the installation and management of Kubernetes clusters, including high availability setups.

```bash
curl -sfL https://get-kk.kubesphere.io | VERSION={versión} sh -
```

To configure the cluster configuration it requires a yaml definition with all IP nodes and his respective roles.

```yaml
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  # Nodes that will make up the cluster, it is necessary to connect the script via SSH
  - {name: master1, address: 192.168.2.51, internalAddress: 192.168.2.51, user: master1, password: ""}
  - {name: master2, address: 192.168.2.52, internalAddress: 192.168.2.52, user: master2, password: ""}
  - {name: master3, address: 192.168.2.53, internalAddress: 192.168.2.53, user: master3, password: ""}
  - {name: worker1, address: 192.168.2.61, internalAddress: 192.168.2.61, user: worker1, password: ""}
  - {name: worker2, address: 192.168.2.62, internalAddress: 192.168.2.62, user: worker2, password: ""}
  - {name: worker3, address: 192.168.2.63, internalAddress: 192.168.2.63, user: worker3, password: ""}
  roleGroups:
    etcd:
    # Control plane nodes alias
    - master1
    - master2
    - master3
    control-plane:
    - master1
    - master2
    - master3
    worker:
    # Worker nodes alias
    - worker1
    - worker2
    - worker3
  controlPlaneEndpoint:
    domain: cluster.noprod.local
    # Virtual IP address created previously
    address: "192.168.2.60"
    # Port defined within HaProxy configuration
    port: 7443
  kubernetes:
    version: v1.23.10
    clusterName: cluster.noprod
    autoRenewCerts: true
    # Container orchestrator
    containerManager: docker
  etcd:
    type: kubekey
  network:
    # Kubernetes network plugin 
    plugin: flannel
    # Virtual subnet that will use the cluster. It must not conflict with other subnets (in my case, 192.168.2.0/24).
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

To deploy the cluster use the Kube Key CLI, it will use ssh to configure all the nodes:

```bash
chmod +x kk
./kk create cluster -f {kube key cluster configuration}.yaml
```

### Persistence of data

Persistence of data is a challenging aspect in distributed systems. I have tested many solutions, and the one I chose is to keep the data replicated. In this approach, data is treated like a pod: you configure a desired number of replicas and the 
system ensures it stays replicated across worker nodes disks. If one node goes down, a persistent volume will reattach the data from another available node while the healthy data is cloned. That is why an extra 1TB SSD has been added to each 
worker node. All of this is achieved using Longhorn, steps to install it:

1. Disable multipath on each worker node

```bash
vi /etc/multipath.conf
```

```plaintext
blacklist {
    devnode "^sd[a-z0-9]+"
}
```

```bash
systemctl enable multipathd
systemctl start multipathd
multipath -ll # To check if mounts are working with multipath
```

2. Longhorn installation:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

3. Worker node disks configuration

```bash
lsblk
fdisk /dev/vdb # n, p, 1, w
mkfs.ext4 /dev/<SSD disk name>
mkdir /mnt/ssd-1 # Can be set orher name
mount /dev/vdb1 /mnt/ssd-1
vi /etc/fstab # Append the next line to /etc/fstab -> "/dev/vdb1 /mnt/ssd-1 ext4 defaults 0 0"
df -h
```

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/longhorn_1.png" >}}

4. Attach Longhorn storage class to PVC

To use longhorn storage class add the next definition into the PVC and PV:

```yaml
# PV manifest example
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
# PVC manifest example
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

Volumes can be created by demand or manually from longhorn UI.


### Exposing services

In the previous guide, we created a cool HA Kubernetes cluster. Nevertheless, without exposing services, it's not very useful.

There are two main options to expose services outside the cluster: via NodePort and LoadBalancer. Using a NodePort doesn't make much sense in an HA setup because the service is tied to a single node, which can be a single point of failure. One 
approach could be to set up a custom load balancer with HAProxy and KeepAlived using a dedicated pool of machines. While that could work, it also means higher costs due to the need for at least two additional servers. Instead, a better option
might be to use the worker nodes themselves as load balancers, leveraging OpenELB.

I chose [OpenELB Layer 2](https://openelb.io/docs/concepts/layer-2-mode/) mode to assign a virtual IP to an Ingress. Here's how it works:
- OpenELB assigns a virtual IP from the real subnet (not from the Kubernetes overlay network) to the service
- The router adds the IP to its ARP table
- The virtual IP is bound to a worker node. If that node goes down, OpenELB reassigns the IP to another healthy node

OpenELB can be installed easily using Kubernetes manifests.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
kubectl apply -f https://raw.githubusercontent.com/openelb/openelb/master/deploy/openelb.yaml
kubectl -n openelb-system get pod
```

The subnet used by OpenELB to assign virtual IPs must be defined using Eip CRD:

```yaml
apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
  name: layer2-eip
spec:
  # Real subnet
  address: 192.168.2.80-192.168.2.200 
  # Physical interface of worker nodes
  interface: enp6s0
  protocol: layer2
```

Now the LoadBalancer service can be created. Once it is deployed, OpenELB will assign an IP from the previously defined subnet range.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: layer2-svc
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
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

In this case, it has set {{< mathematics/inline >}}192.168.2.80{{< /mathematics/inline >}} as the external IP because it's the first address in the EIP address range.

{{< images/centered-image src="/posts/how-my-on-premise-private-cloud-was-built/openelb_1.png" >}}

The last step to achieve a solid solution is to secure access to the Ingress using a custom CA. It can be done with cert-manager, a built-in Kubernetes service that orchestrates certificates.

It requires creating a key to sing certificates with the CA.

```yaml
# Creates the private key
openssl genrsa -out ca.key 4096

# Signs the certificate
openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt
```

cert-manager store CA certificate through a ClusterIssuer CRD.

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
  tls.crt: # Base64-encoded content of ca.crt (e.g., cat ca.crt | base64 -w 0)
  tls.key: # Base64-encoded content of ca.key (e.g., cat ca.key | base64 -w 0)
```

To expose the service outside the cluster, we have to combine Nginx and cert-manager within a kubernetes Ingress manifest.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # This annotation tells cert-manager to use the specified ClusterIssuer (in this case, a CA-based issuer)
    # to automatically issue a TLS certificate for this Ingress.
    cert-manager.io/cluster-issuer: ca-cluster-issuer
  name: gitea-ha-gitea-ha
  namespace: infra
spec:
  # Specifies that this Ingress should be handled by the NGINX Ingress Controller
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
    # cert-manager will store the issued TLS certificate in this secret.
    # If it doesn't exist, cert-manager will create it and renew it automatically.
    - pre.gitea.com
    secretName: gitea-ha-gitea-ha-tls
```

# Chapter 3: The solution becomes the problem

Following this guideline, it almost feels like anyone could build a basic HA Kubernetes cluster with a pool of toasters. Looks easy, right? But I made it clear at the beginning of this post: creating and maintaining a Kubernetes cluster is hard—especially
when it's hosted on-premise with as few resources as possible. I’ve had to deal with a lot of problems just to keep the environment running—most of them the result of my own bad decisions.

At one point, the worker nodes started rebooting randomly with a kernel memory segmentation exception. I spent a lot of time trying to figure out the cause of the error. After a few weeks without finding the origin of the issue, I threw in the towel. I
decided to copy the data from the persistent volumes to my personal computer and reinstall the entire cluster. But... it showed the same error. I started thinking about all the changes I had made recently, and then I realized the real issue: a RAM slot 
I had added not long ago. It was running at a different frequency than the others. Yes, I spent a lot of time... and the fix was embarrassingly simple.

Another issue was using a Molex-to-SATA connection. In the beginning, before installing Longhorn, all the data was stored on a separate host. It acted as the Kubernetes storage controller using NFS. The motherboard on that host only had six SATA slots,
so I used a Molex-to-SATA adapter to increase the number of disks in the RAID array. Eventually, one of the drives in the array failed, and I’m pretty sure the cause was the unstable power provided by that Molex connection. It seemed like a practical
workaround at the time, but it ended up compromising the reliability of the whole setup.

# Chapter 4: Conclusions

As an engineer, you must carefully evaluate whether an on-premise Kubernetes cluster is the best approach to achieve your goals. The list of problems is endless, and dealing with them drains a lot of time and energy. Nevertheless, if you decide to
build one, the knowledge you gain will be incredibly valuable. The lessons learned from working through the challenges of managing your own cluster are indispensable and will help shape your decision-making in future projects.
