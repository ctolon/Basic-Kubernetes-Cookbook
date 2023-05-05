# Kubernetes Prod Env. Setup Guide

- [Kubernetes Prod Env. Setup Guide](#kubernetes-prod-env-setup-guide)
  - [Pre-requirements](#pre-requirements)
    - [1. Install Docker with Docker Engine](#1-install-docker-with-docker-engine)
    - [Opsiyonel: Install VM with multipass](#opsiyonel-install-vm-with-multipass)
  - [Core Setup](#core-setup)
    - [1. Configure IP Settings](#1-configure-ip-settings)
    - [2. Optional: Install Containerd](#2-optional-install-containerd)
    - [2. Optional: Install CRI Docker](#2-optional-install-cri-docker)
    - [3. Swapoff Settings](#3-swapoff-settings)
    - [Optional: Install Specific version of kustomize](#optional-install-specific-version-of-kustomize)
    - [4. Install Kubernetes tools](#4-install-kubernetes-tools)
    - [5. Install Kubernetes Cluster](#5-install-kubernetes-cluster)


## Pre-requirements

### 1. Install Docker with Docker Engine

```bash
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
    
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg


echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update
sudo chmod a+r /etc/apt/keyrings/docker.gpg
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
sudo chmod g+rwx "$HOME/.docker" -R
sudo apt-get install git-all
sudo chmod 666 /var/run/docker.sock
```

### Opsiyonel: Install VM with multipass

Burada her ihtimale karşı resolvconf'i default ayarlarına sıfırlıyoruz. VM yapılandırması için multipass'ı kuruyoruz. driver olarak lxd atıyoruz (linux için çalışan tek driver), multipass networks ile VM'in network ayarları doğru set edilmiş mi kontrol ediyoruz. Ardından master ve worker nodelarımızı init ediyoruz. bunları init ettikten sonra ayrı shellerde `multipass shell <node>` içlerine giriyoruz.


```bash
# VM Settings for cluster management
sudo apt-get install --reinstall resolvconf
sudo snap install multipass
multipass set local.driver=lxd
multipass networks
multipass launch --name master -c 2 -m 2G -d 10G
multipass launch --name node1 -c 2 -m 2G -d 10G

# On different shells
multipass shell master
multipass shell node1

# You can look at ip of VM with shell master.
```

## Core Setup

### 1. Configure IP Settings

Kubernetes için IP ayarlarını set ediyoruz.

```bash
# Iptables bridged traffic config
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo su
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
systemctl restart systemd-modules-load.service
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6table
exit

sudo su
modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
sudo sysctl -p
exit


# Apply sysctl params without reboot
sudo sysctl --system

lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 2. Optional: Install Containerd

VM içinde çalışırsak containerd'yi run time olarak ayarlıyoruz:

```bash
### Install Containerd ### 

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install containerd -y
sudo mkdir -p /etc/containerd
sudo su -
containerd config default | tee /etc/containerd/config.toml
exit
sudo systemctl restart containerd

	
sudo ufw allow 6443
sudo ufw allow 6443/tcp
```

### 2. Optional: Install CRI Docker

VM içinde alternatif olarak CRI Dockerı run time olarak ayarlayabiliriz. Ama zaten VM set ettiğimiz için buna gerek yok, ayrıca bu aşama için VM'ler içinde docker olması kesin lazım:

```bash
git clone https://github.com/Mirantis/cri-dockerd.git

--> NOTE: Run these commands as root

wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile

cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
cp -a packaging/systemd/* /etc/systemd/system
sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl enable cri-docker.service
systemctl enable --now cri-docker.socket
```

### 3. Swapoff Settings

Kubernetes'in düzgün çalışması için swap-off ayarları:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

sudo -i
swapoff -a
exit

# confirmation
sudo swapoff -a
sudo mount -a
free -h

```

### Optional: Install Specific version of kustomize

Opsiyonal versiyonla kustomize kurulumu:

```bash
curl --silent --location --remote-name \
"https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize/v5.0.1/kustomize_v5.0.1_linux_amd64.tar.gz" && \
tar -xf kustomize_v5.0.1_linux_amd64.tar.gz && \
chmod a+x kustomize && \
sudo mv kustomize /usr/local/bin/kustomize
```


### 4. Install Kubernetes tools

Burada kubernetes toollarını kuruyoruz:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

# With Latest Version
sudo apt-get install -y kubelet kubeadm kubectl

# With Provided Version (ex. 1.25)
sudo apt -y install vim git curl wget kubelet=1.25.8-00 kubeadm=1.25.8-00 kubectl=1.25.8-00

sudo apt-mark hold kubelet kubeadm kubectl
```

En son configler ile kubernetes clusterımızı ayağa kaldırıyoruz:

### 5. Install Kubernetes Cluster

Burada kubernetes clusterı için önce imageları pull ediyoruz, sonrasında ise kubeadm init ile clusterı command line üzerinden konfigüre ederek ayağa kaldırabiliyoruz. Template şu şekilde:

```bash
sudo kubeadm init --pod-network-cidr=<cidr> --apiserver-advertise-address=<api-id> --control-plane-endpoint=<cpe> --kubernetes-version <kv> --v 5 --cri-socket=<socket>
```

Parametreler:

* `cidr`: Network ayarlarında flannel kullanılacaksa 10.244.0.0/16, calico için 192.168.0.0/16

* `cpe`: control plane endpoint, bu master node'un ipsi ile port değerini alır. default olarak port 6443'tür.

* `kv`: Kubernetes için indirilecek image'ların versiyonunu opsiyonel olarak tanımlayabiliriz. kubelet, kubeadm ve kubectl nin versiyonundan bağımsız kubernetes imageları için versiyon seçmeye yarar.

* `socket`: Container run time interface için socket seçmeye yarar. Eğer birden fazla soket kuruluysa hem init hem reset için kesinlikle konfigüre edilmelidir.

* `api-id`: Api server ip


Clusterı ayağa kaldıran configlerden bazı örnekler

```bash

# Only for master node
sudo kubeadm config images pull --cri-socket=unix:///var/run/cri-dockerd.sock

# With CRI Containerd # Only for master node
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip> --kubernetes-version 1.25.8 --v 5 --cri-socket=unix:///var/run/cri-dockerd.sock

sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket unix:///run/cri-dockerd.sock --apiserver-advertise-address=192.168.2.105 --control-plane-endpoint 192.168.2.105:6443


# With Default Containerd # Only for master node
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<ip> --control-plane-endpoint=<ip> --kubernetes-version 1.25.0 --v 5 --cri-socket=<unix::containerd>

# For flannel
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.2.105 --control-plane-endpoint=192.168.2.105:6443 --kubernetes-version 1.25.0 --v 5 --cri-socket=unix:///var/run/crio/crio.sock

sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.244.0.0 --control-plane-endpoint=10.244.0.0:6443 --kubernetes-version 1.25.0 --v 5 --cri-socket=unix:///var/run/crio/crio.sock


# Only for worker node
sudo kubeadm join 10.32.227.100:6443 --token jsr3p8.2175i23pvmm7qyur \
	--discovery-token-ca-cert-hash sha256:4bfde2437ab88faa22a275d6c01ed9312728634bb903eaf4fe27dd9467834434 

# ip: server ip address (flannel için cidr 10.244.0.0/16 olmalı)

# only for master
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# ONLY FOR MASTER
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# only for worker
kubectl create -f https://downloads.tigera.io/ee/v3.15.2/manifests/tigera-operator.yaml
kubectl create -f https://downloads.tigera.io/ee/v3.15.2/manifests/custom-resources.yaml	
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```



`TODO`: Burayı düzenle

sudo kubeadm init --upload-certs --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.3.12 --control-plane-endpoint 192.168.3.12:6443 --kubernetes-version=1.25.0 --v 5 --cri-socket=unix:///var/run/containerd/containerd.sock

sudo kubeadm reset --v 5 --cri-socket=unix:///var/run/containerd/containerd.sock


kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.3.12 --v 5 --cri-socket=/var/run/crio/crio.sock


    go get -d github.com/containernetworking/plugins
    cd ~/go/src/github.com/containernetworking/plugins
    ./build.sh
    sudo cp bin/* /opt/cni/bin/

    go mod tidy
    go mod vendor

https://stackoverflow.com/questions/53900779/pods-failed-to-start-after-switch-cni-plugin-from-flannel-to-calico-and-then-fla


