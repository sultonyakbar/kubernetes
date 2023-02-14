# Setup Kubernetes Multi Master (Master/Controlplane Multi Master)

## Persiapan

* 1 VM Load Balancer Master/Controlplan dengan spek 2vcpu/4GB RAM
* 3 VM Master/Controlplan dengan spek 2vcpu/4GB RAM
* 3 VM Worker dengan spek 2vcpu/8GB RAM (bisa lebih gede/kecil tergantung aplikasi yang akan dideploy)
* Seluruh VM Master/Worker disable swap memory permanent
* Security Group antar node Master/Worker di open-all (kecuali anda sudah punya list port2 service kubernetes)


## List 

|        Nama          |      IP      |       OS      | 
|----------------------|--------------|---------------|
| Load Balancer Master | 172.16.10.10 |  Ubuntu 18.04 |
| Master/Control Plane | 172.16.10.11 |  Ubuntu 18.04 |
| Master/Control Plane | 172.16.10.12 |  Ubuntu 18.04 |
| Master/Control Plane | 172.16.10.13 |  Ubuntu 18.04 |
|     Worker-01        | 172.16.10.20 |  ubuntu 18.04 |
|     Worker-02        | 172.16.10.21 |  ubuntu 18.04 |
|     Worker-03        | 172.16.10.23 |  ubuntu 18.04 |

## Konfigurasi Load Balancer Master/Control Plan
### Setup Nginx
```
load-balancer-master~# apt-get update -y
load-balancer-master~# apt-get install nginx -y
```

Tambahkan  pada file `/etc/nginx/nginx.conf` di baris paling bawah
```
stream{
	upstream master_conn{
		server 172.16.10.11:6443;
		server 172.16.10.12:6443;
		server 172.16.10.13:6443;
	}
	server {
	listen 6443;
	proxy_pass master_conn;
  }
}
```
Reload Konfigurasi Nginx
```
load-balancer-master~# nginx -s reload
````

## Konfigurasi Master/Control Plan
### Setup Containerd
```
master-01~# swapoff -a
master-01~# cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
master-01~# modprobe overlay
master-01~# modprobe br_netfilter
master-01~# cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
master-01~# sysctl --system
master-01~# apt-get update -y
master-01~# apt-get install apt-transport-https ca-certificates curl software-properties-common lsb-release -y
master-01~# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
master-01~# echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
master-01~# apt-get update && apt-get install containerd.io -y
master-01~# mkdir -p /etc/containerd
master-01~# containerd config default | sudo tee /etc/containerd/config.toml
master-01~# systemctl restart containerd
master-01~# systemctl enable containerd
```

Tambahkan :
```
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
``` 
pada file `/etc/containerd/config.toml` didalam block `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]`
```
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v1"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
```
master-01~# systemctl restart containerd
```
Lakukan step by step konfigurasi master pada seluruh node master

### Setup Tools Kubernetes
```
master-01~# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
master-01~# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
master-01~# apt-get update -y
master-01~# apt-get install kubelet kubeadm kubectl -y
master-01~# apt-mark hold kubeadm kubectl kubelet containerd.io

```
* NB: Jika butuh spesifik versi kubernetes. tambahkan seperti ini `kubelet=1.19.7-00 kubeadm=1.19.7-00 kubectl=1.19.7-00`
* Check versi kubernetes `apt-cache madison kubeadm`
* Lakukan step by step Setup Tools Kubernetes pada seluruh node master

### Konfigurasi Sertifikat Etcd
```
master-01~# mkdir ca && cd ca
master-01~# cat <<EOF | sudo tee ca-config.json
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
master-01~# cat <<EOF | sudo tee ca-csr.json
{
    "CN": "MYID CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "ID",
            "L": "DKI Jakarta",
            "O": "IT",
            "ST": "Jakarta Pusat",
            "OU": "Team Infra"
        }
    ]
}
EOF
master-01~# cat <<EOF | sudo tee kubernetes-csr.json
{
    "CN": "MYID CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "ID",
            "L": "DKI Jakarta",
            "O": "IT",
            "ST": "Jakarta Pusat",
            "OU": "Team Infra"
        }
    ]
}
EOF
master-01~# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
master-01~# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
master-01~# chmod +x cfssl*
master-01~# sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
master-01~# sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
master-01~# cfssl version
master-01~# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
master-01~# cfssl gencert \
-ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-hostname=172.16.10.10,172.16.10.11,172.16.10.12,172.16.10.13,127.0.0.1 \
-profile=kubernetes kubernetes-csr.json | \
cfssljson -bare kubernetes
```
Hasil yang didapatkan dari generate sertifikat untuk master adalah ini:
```
ca.pem
kubernetes.pem
kubernetes-key.pem
master-01~# sudo mkdir /etc/etcd /var/lib/etcd
master-01~# sudo mv ca.pem kubernetes.pem kubernetes-key.pem /etc/etcd
```
* NB: Copy ketiga file tersebut ke seluruh node master dan pindahkan ke folder /etc/etcd

### Setup Manual Etcd
#### node master-01

```
master-01~# sudo wget https://github.com/etcd-io/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz
master-01~# sudo tar xvzf etcd-v3.4.3-linux-amd64.tar.gz
master-01~# mv etcd-v3.4.3-linux-amd64/etcd* /usr/local/bin/.
master-01~# sudo cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name 172.16.10.11 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://172.16.10.11:2380 \
  --listen-peer-urls https://172.16.10.11:2380 \
  --listen-client-urls https://172.16.10.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.10.11:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 172.16.10.11=https://172.16.10.11:2380,172.16.10.12=https://172.16.10.12:2380,172.16.10.13=https://172.16.10.13:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
master-01~# systemctl daemon-reload
master-01~# sudo systemctl enable etcd
master-01~# sudo systemctl start etcd
```
#### node master-02
```
master-02~# sudo wget https://github.com/etcd-io/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz
master-02~# sudo tar xvzf etcd-v3.4.3-linux-amd64.tar.gz
master-02~# mv etcd-v3.4.3-linux-amd64/etcd* /usr/local/bin/.
master-02~# sudo cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name 172.16.10.12 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://172.16.10.12:2380 \
  --listen-peer-urls https://172.16.10.12:2380 \
  --listen-client-urls https://172.16.10.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.10.12:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 172.16.10.11=https://172.16.10.11:2380,172.16.10.12=https://172.16.10.12:2380,172.16.10.13=https://172.16.10.13:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
master-02~# systemctl daemon-reload
master-02~# sudo systemctl enable etcd
master-02~# sudo systemctl start etcd
```
### node master-03
```
master-03~# sudo wget https://github.com/etcd-io/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz
master-03~# sudo tar xvzf etcd-v3.4.3-linux-amd64.tar.gz
master-03~# mv etcd-v3.4.3-linux-amd64/etcd* /usr/local/bin/.
master-03~# sudo cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name 172.16.10.13 \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://172.16.10.13:2380 \
  --listen-peer-urls https://172.16.10.13:2380 \
  --listen-client-urls https://172.16.10.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://172.16.10.13:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster 172.16.10.11=https://172.16.10.11:2380,172.16.10.12=https://172.16.10.12:2380,172.16.10.13=https://172.16.10.13:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
master-03~# systemctl daemon-reload
master-03~# sudo systemctl enable etcd
master-03~# sudo systemctl start etcd
```
* NB : nyalakan service etcd cluster 1 by 1

### Setup Kubernetes Cluster
```
master-01~#cat <<EOF | sudo tee kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
apiServer:
  certSANs:
  - "172.16.10.10"
controlPlaneEndpoint: "172.16.10.10:6443"
etcd:
  external:
    endpoints:
    - https://172.16.10.11:2379
    - https://172.16.10.12:2379
    - https://172.16.10.13:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.30.0.0/16
nodeRegistration:
  criSocket: /run/containerd/containerd.sock

---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

```
master-01~# kubeadm init --config kubeadm-config.yaml --upload-certs
```

Output yang akan didapatkan adalah command untuk join node master, contoh seperti ini :
```
kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef --control-plane --certificate-key aabbccddee11223344 172.16.10.10:6443 --cri-socket=/run/containerd/containerd.sock
```
Output yang akan didapatkan adalah command untuk join node worker, contoh seperti ini :
```
kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef 172.16.10.10:6443 --cri-socket=/run/containerd/containerd.sock
```

Lakukan join node master pada node master-02 & master-03 :
```
master-02~# kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef --control-plane --certificate-key aabbccddee11223344 172.16.10.10:6443 --cri-socket=/run/containerd/containerd.sock
master-03~# kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef --control-plane --certificate-key aabbccddee11223344 172.16.10.10:6443 --cri-socket=/run/containerd/containerd.sock
```

## Konfigurasi Worker
### Setup Containerd
```
worker-01~# swapoff -a
worker-01~# cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
worker-01~# modprobe overlay
worker-01~# modprobe br_netfilter
worker-01~# cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
worker-01~# sysctl --system
worker-01~# apt-get update -y
worker-01~# apt-get install apt-transport-https ca-certificates curl software-properties-common lsb-release -y
worker-01~# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
worker-01~# echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
worker-01~# apt-get update && apt-get install containerd.io -y
worker-01~# mkdir -p /etc/containerd
worker-01~# containerd config default | sudo tee /etc/containerd/config.toml
worker-01~# systemctl restart containerd
worker-01~# systemctl enable containerd
```
Tambahkan :
```
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
``` 
pada file `/etc/containerd/config.toml` didalam block `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]`
```
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v1"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```
```
worker-01~# systemctl restart containerd
```

* NB: Versi Containerd harus sama dengan versi dinode Master/controlplane

### Setup Tools Kubernetes
```
worker-01~# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
worker-01~# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
worker-01~# apt-get update -y
worker-01~# apt-get install kubelet kubeadm kubectl -y
worker-01~# apt-mark hold kubeadm kubectl kubelet containerd.io
```
* NB: Versi kubelet,kubectl,kubeadm harus sama dengan versi dinode Master/Controlplane

Paste Command Join worker yang didapatkan setelah setup di node master :

`worker-01~# kubeadm join --discovery-token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:1234..cdef 172.16.10.10::6443 --cri-socket=/run/containerd/containerd.sock`


Finish, lakukan step by step konfigurasi worker pada seluruh node worker



### Setup CNI Kubernetes
Jika semua worker sudah Join, langkah terakhir adalah menginstall CNI Kubernetes. ada banyak project list CNI kubernetes yang bisa digunakan, disini kita akan menggunakan CNI Calico.


`kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml`


To start using your cluster, you need to run the following as a regular user:


Finish, sekarang bisa check status node master/worker

 ` master-01~$ mkdir -p $HOME/.kube` 
 
 ` master-01~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config` 
 
 ` master-01~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config` 
 
` master-01~$ kubectl get nodes`





