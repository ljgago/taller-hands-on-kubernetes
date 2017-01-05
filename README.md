# taller-hands-on-kubernetes

Taller para instalar kubernetes (paso a paso, the hard way) en un cluster de coreOS en tu Laptop.

## Preliminares

Lo primero es clonar este repositorio, y levantar las máquinas

> **git clone https://github.com/tidchile/taller-hands-on-kubernetes**  
> **cd taller-hands-on-kubernetes**  
> **vagrant up**  
> **vagrant status**  

Las máquinas están arriba!

Una muy buena aclaración es que porque estemos usando Vagrant, no significa que
necesitemos Vagrant para kubernetes: Simplemente es una forma cómoda de levantar
un número de máquinas virtuales en tu laptop. El escenario ideal es tener los
servidores remotos. Que es donde debería correr la solución de kubernetes.

## ETCD Server

Entramos a la primera máquina. Esta es la que designaremos para correr kubernetes.

> **vagrant ssh core-01**  
> **ifconfig # Nos entregaría la "IP Pública de la maquina, en este ejemplo será 172.17.8.101"**  

Creamos el drop-in

> **sudo mkdir -p /etc/systemd/system/etcd2.service.d**  
> **sudo vim /etc/systemd/system/etcd2.service.d/40-listen-address.conf**  

Dentro del editor:

```
[Service]
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://172.17.8.101:2379
```

Levantemos el servicio, dejemoslo amarrado al startup y revisemoslo

> **sudo systemctl daemon-reload**  
> **sudo systemctl start etcd2**  
> **sudo systemctl enable etcd2**  
> **systemctl status etcd2**  

Para comprobar si funciona de afuera

> **curl http://172.17.8.101:2379/v2/keys**  

Recuerden salir de la máquina core-01 para seguir ejecutando las siguientes sentencias.

## Generemos los elementos TLS

Necesitamos conocer _a priori_ la IP del servidor que escojamos como master.

Usemos el script de este mismo repositorio. Supongamos que la IP de master es
`172.17.8.102`, haremos

> **./hack/generate-tls-assets.sh 172.17.8.102**  

## Master

### SSL

Movamos los siguientes archivos a la maquina master

> **scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,apiserver,apiserver-key}.pem core@172.17.8.102:.**  

Y dentro de la maquina master

> **sudo mkdir -p /etc/kubernetes/ssl**  
> **sudo mv {ca,apiserver,apiserver-key}.pem /etc/kubernetes/ssl/**  
> **sudo chmod 600 /etc/kubernetes/ssl/*-key.pem**  
> **sudo chown root:root /etc/kubernetes/ssl/*-key.pem**

### Flannel

> **sudo mkdir /etc/flannel**  
> **sudo vim /etc/flannel/options.env**

Dentro del editor

```
FLANNELD_IFACE=${ADVERTISE_IP}
FLANNELD_ETCD_ENDPOINTS=${ETCD_ENDPOINTS}
```

```
FLANNELD_IFACE=172.17.8.102
FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379
```

> **sudo mkdir /etc/systemd/system/flanneld.service.d**  
> **sudo vim /etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf**

```
[Service]
ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
```

### Docker

```bash
sudo mkdir /etc/systemd/system/docker.service.d
sudo vim /etc/systemd/system/docker.service.d/40-flannel.conf
```

```
[Unit]
Requires=flanneld.service
After=flanneld.service
[Service]
EnvironmentFile=/etc/kubernetes/cni/docker_opts_cni.env
```

Creo un archivo de Opciones CNI para Docker:

> **sudo mkdir /etc/kubernetes/cni**  
> **sudo vim /etc/kubernetes/cni/docker_opts_cni.env**

```bash
DOCKER_OPT_BIP=""
DOCKER_OPT_IPMASQ=""
```

Si usamos Flannel para networking, configuramos el Flannel CNI:

> **sudo mkdir /etc/kubernetes/cni/net.d/**  
> **sudo vim /etc/kubernetes/cni/net.d/10-flannel.conf**

```json
{
    "name": "podnet",
    "type": "flannel",
    "delegate": {
        "isDefaultGateway": true
    }
}
```

### Kubernetes Services

**sudo vim /etc/systemd/system/kubelet.service**

```
[Service]
Environment=KUBELET_VERSION=v1.5.1_coreos.0
Environment="RKT_OPTS=--uuid-file-save=/var/run/kubelet-pod.uuid \
  --volume var-log,kind=host,source=/var/log \
  --mount volume=var-log,target=/var/log \
  --volume dns,kind=host,source=/etc/resolv.conf \
  --mount volume=dns,target=/etc/resolv.conf"
ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
ExecStartPre=/usr/bin/mkdir -p /var/log/containers
ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
ExecStart=/usr/lib/coreos/kubelet-wrapper \
  --api-servers=https://127.0.0.1:8080 \
  --register-schedulable=false \
  --cni-conf-dir=/etc/kubernetes/cni/net.d \
  --network-plugin=cni \
  --container-runtime=docker \
  --allow-privileged=true \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --hostname-override=172.17.8.102 \
  --cluster_dns=10.3.0.10 \
  --cluster_domain=cluster.local
ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> **sudo mkdir -p /etc/kubernetes/manifests**

**sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml**  

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-apiserver
    image: quay.io/coreos/hyperkube:v1.5.1_coreos.0
    command:
    - /hyperkube
    - apiserver
    - --bind-address=0.0.0.0
    - --etcd-servers=http://172.17.8.101:2379
    - --allow-privileged=true
    - --service-cluster-ip-range=10.3.0.0/24
    - --secure-port=443
    - --advertise-address=172.17.8.102
    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --runtime-config=extensions/v1beta1/networkpolicies=true
    - --anonymous-auth=false
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        port: 8080
        path: /healthz
      initialDelaySeconds: 15
      timeoutSeconds: 15
    ports:
    - containerPort: 443
      hostPort: 443
      name: https
    - containerPort: 8080
      hostPort: 8080
      name: local
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

**sudo vim /etc/kubernetes/manifests/kube-proxy.yaml**  

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.5.1_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=http://127.0.0.1:8080
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  volumes:
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

**sudo vim /etc/kubernetes/manifests/kube-controller-manager.yaml**  

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-controller-manager
    image: quay.io/coreos/hyperkube:v1.5.1_coreos.0
    command:
    - /hyperkube
    - controller-manager
    - --master=http://127.0.0.1:8080
    - --leader-elect=true
    - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
    resources:
      requests:
        cpu: 200m
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10252
      initialDelaySeconds: 15
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/kubernetes/ssl
      name: ssl-certs-kubernetes
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ssl-certs-host
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/ssl
    name: ssl-certs-kubernetes
  - hostPath:
      path: /usr/share/ca-certificates
    name: ssl-certs-host
```

**sudo vim /etc/kubernetes/manifests/kube-scheduler.yaml**  

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-scheduler
    image: quay.io/coreos/hyperkube:v1.5.1_coreos.0
    command:
    - /hyperkube
    - scheduler
    - --master=http://127.0.0.1:8080
    - --leader-elect=true
    resources:
      requests:
        cpu: 100m
    livenessProbe:
      httpGet:
        host: 127.0.0.1
        path: /healthz
        port: 10251
      initialDelaySeconds: 15
      timeoutSeconds: 15
```

OK. Echemos a andar master!

> **sudo systemctl daemon-reload**  
> 
> **curl -X PUT -d "value={\"Network\":\"10.2.0.0/16\",\"Backend\":{\"Type\":\"vxlan\"}}" "http://172.17.8.101:2379/v2/keys/coreos.com/network/config"** 
> 
> **sudo systemctl start flanneld**  
> **sudo systemctl enable flanneld**  
> 
> **sudo systemctl start kubelet**  
> **sudo systemctl enable kubelet**  

Comprobemos si anda

> **systemctl status kubelet**  
> **docker ps -a**  

Antes de crear el namespace `kube-system`, veamos si anda la API de kubernetes:

> **curl http://127.0.0.1:8080/version**  

```json
{
  "major": "1",
  "minor": "5",
  "gitVersion": "v1.5.1+coreos.0",
  "gitCommit": "cc65f5321f9230bf9a3fa171155c1213d6e3480e",
  "gitTreeState": "clean",
  "buildDate": "2016-12-14T04:08:28Z",
  "goVersion": "go1.7.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Ya. Creemos el namespace

> **curl -H "Content-Type: application/json" -XPOST -d'{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"kube-system"}}' "http://127.0.0.1:8080/api/v1/namespaces"**  

Podemos revisar los pods creados por medio de los medatados de la API:

> **curl -s localhost:10255/pods | jq -r '.items[].metadata.name'**  

## Workers

### TLS

Movamos para cada maquina los archivos

> **scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,worker,worker-key}.pem core@172.17.8.103:.**  
> **scp -i ~/.vagrant.d/insecure_private_key secrets/{ca,worker,worker-key}.pem core@172.17.8.104:.**  

Y dentro de cada maquina

> **sudo mkdir -p /etc/kubernetes/ssl**  
> **sudo mv {ca,worker,worker-key}.pem /etc/kubernetes/ssl/**  
> **sudo chmod 600 /etc/kubernetes/ssl/*-key.pem**  
> **sudo chown root:root /etc/kubernetes/ssl/*-key.pem**  

### Flannel

> **sudo mkdir /etc/flannel**  
> **sudo vim /etc/flannel/options.env**  

```
FLANNELD_IFACE=172.17.8.103
FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379
```

> **sudo mkdir /etc/systemd/system/flanneld.service.d**  
> **sudo vim /etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf**  

```
[Service]
ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
```

### Docker

> **sudo mkdir /etc/systemd/system/docker.service.d**  
> **sudo vim /etc/systemd/system/docker.service.d/40-flannel.conf** 

```
[Unit]
Requires=flanneld.service
After=flanneld.service
[Service]
EnvironmentFile=/etc/kubernetes/cni/docker_opts_cni.env
```

Creo el archivo de opciones Docker CNI:

> **sudo mkdir /etc/kubernetes/cni**  
> **sudo vim /etc/kubernetes/cni/docker_opts_cni.env**  

```
DOCKER_OPT_BIP=""
DOCKER_OPT_IPMASQ=""
```

Configuro le Flannel CNI:

> **sudo mkdir /etc/kubernetes/cni/net.d/**  
> **sudo vim /etc/kubernetes/cni/net.d/10-flannel.conf**  

```json
{
    "name": "podnet",
    "type": "flannel",
    "delegate": {
        "isDefaultGateway": true
    }
}
```

### Kubelet  

**sudo vim /etc/systemd/system/kubelet.service**  

```
[Service]
Environment=KUBELET_VERSION=v1.5.1_coreos.0
Environment="RKT_OPTS=--uuid-file-save=/var/run/kubelet-pod.uuid \
  --volume dns,kind=host,source=/etc/resolv.conf \
  --mount volume=dns,target=/etc/resolv.conf \
  --volume var-log,kind=host,source=/var/log \
  --mount volume=var-log,target=/var/log"
ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
ExecStartPre=/usr/bin/mkdir -p /var/log/containers
ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
ExecStart=/usr/lib/coreos/kubelet-wrapper \
  --api-servers=https://172.17.8.102 \
  --cni-conf-dir=/etc/kubernetes/cni/net.d \
  --network-plugin=cni \
  --container-runtime=docker \
  --register-node=true \
  --allow-privileged=true \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --hostname-override=172.17.8.103 \
  --cluster_dns=10.3.0.10 \
  --cluster_domain=cluster.local \
  --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml \
  --tls-cert-file=/etc/kubernetes/ssl/worker.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/worker-key.pem
ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> **sudo mkdir -p /etc/kubernetes/manifests** 

**sudo vim /etc/kubernetes/manifests/kube-proxy.yaml**  

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-proxy
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
  - name: kube-proxy
    image: quay.io/coreos/hyperkube:v1.5.1_coreos.0
    command:
    - /hyperkube
    - proxy
    - --master=https://172.17.8.102
    - --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: "ssl-certs"
    - mountPath: /etc/kubernetes/worker-kubeconfig.yaml
      name: "kubeconfig"
      readOnly: true
    - mountPath: /etc/kubernetes/ssl
      name: "etc-kube-ssl"
      readOnly: true
  volumes:
  - name: "ssl-certs"
    hostPath:
      path: "/usr/share/ca-certificates"
  - name: "kubeconfig"
    hostPath:
      path: "/etc/kubernetes/worker-kubeconfig.yaml"
  - name: "etc-kube-ssl"
    hostPath:
      path: "/etc/kubernetes/ssl"
```

**sudo vim /etc/kubernetes/worker-kubeconfig.yaml**  

```
apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/ssl/ca.pem
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/ssl/worker.pem
    client-key: /etc/kubernetes/ssl/worker-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context
```

### Start Services

> **sudo systemctl daemon-reload**  
> **sudo systemctl start flanneld**  
> **sudo systemctl start kubelet**  
> **sudo systemctl enable flanneld**  
> **sudo systemctl enable kubelet**  
> **systemctl status kubelet**  

## Kubectl (manejemos la cosa)

kubectl es la magia acá. Te hace "sentir" que no manejas un cluster, sino que te estás entendiendo
con un servidor.
Si usamos la maquina core-01 como administrador:

> **curl -O https://storage.googleapis.com/kubernetes-release/release/v1.5.1/bin/linux/amd64/kubectl**  
> 
> **chmod +x kubectl**  
> **sudo mkdir -p /opt/bin**   
> **sudo mv kubectl /opt/bin/**
> **sudo chown root:root /opt/bin/kubectl**    
> 
> **kubectl config set-cluster default-cluster --server=https://${MASTER_HOST} --certificate-authority=${CA_CERT}**  
> **kubectl config set-credentials default-admin --certificate-authority=${CA_CERT} --client-key=${ADMIN_KEY} --client-certificate=${ADMIN_CERT}**  
> **kubectl config set-context default-system --cluster=default-cluster --user=default-admin**  
> **kubectl config use-context default-system**  

Por ejemplo yo hice, fuera de las máquinas virtuales:

> **cp -i secrets/{ca,admin,admin-key} ~/.ssl/**
> **cd**  
> **kubectl config set-cluster default-cluster --server=https://172.17.8.102 --certificate-authority=.ssl/ca.pem**  
> **kubectl config set-credentials default-admin --certificate-authority=ca.pem --client-key=.ssl/admin-key.pem --client-certificate=.ssl/admin.pem**  
> **kubectl config set-context default-system --cluster=default-cluster --user=default-admin**  
> **kubectl config use-context default-system**  

YMMV of course

### Veamos si funcionó!

> **kubectl get nodes**  
> **kubectl get po --namespace=kube-system**  

## Instalado Addons

### DNS Addon

De aquí en adelante, no deberíamos meternos más (en teoría a los servidores), o sea, jovenes, `kubectl` es tu amigo

> **kubectl create -f dns-addon2.yml**  
> **kubectl get pods --namespace=kube-system | grep kube-dns-v20**  

### Dashboard Addon

> **kubectl create -f kube-dashboard-rc.yaml**  
> **kubectl create -f kube-dashboard-svc.yaml**  

Accedemos al dashboard via port forwarding con `kubectl`.

> **kubectl get pods --namespace=kube-system**  
> **kubectl port-forward kubernetes-dashboard-v1.4.1-SOME-ID 9090 --namespace=kube-system**  

Muchas Gracias.

HJ.-
