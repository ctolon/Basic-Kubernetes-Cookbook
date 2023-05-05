# Kubernetes Test Env. Setup Guide

- [Kubernetes Test Env. Setup Guide](#kubernetes-test-env-setup-guide)
  - [Pre-requirements](#pre-requirements)
    - [1. Install Docker with Docker Engine](#1-install-docker-with-docker-engine)
  - [Core Setup](#core-setup)
    - [1. Configure IP Settings](#1-configure-ip-settings)
    - [2. Install Containerd](#2-install-containerd)
    - [Optional: Install CRI Docker](#optional-install-cri-docker)
    - [3. Swapoff Settings](#3-swapoff-settings)
    - [4. Install Kubernetes tools](#4-install-kubernetes-tools)
    - [5. Install Test Env. With Minikube and Kind](#5-install-test-env-with-minikube-and-kind)


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
    

## Core Setup

### 1. Configure IP Settings

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

# Apply sysctl params without reboot
sudo sysctl --system

lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

### 2. Install Containerd

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
```

### Optional: Install CRI Docker

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

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 4. Install Kubernetes tools

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update

# With Latest Version
sudo apt-get install -y kubelet kubeadm kubectl

# With Provided Version (ex. 1.25)
sudo apt -y install vim git curl wget kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00

sudo apt-mark hold kubelet kubeadm kubectl
```

### 5. Install Test Env. With Minikube and Kind

```bash
# Method 1: Install With kind
go install sigs.k8s.io/kind@v0.17.0
kind create cluster --retain -v 5 --name nm-kubeflow --image kindest/node:v1.21.2

# optional args: --image kindest/node:v1.14.1

# Method 2: Install with minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --cpus 4 --memory 12g --disk-size=80g --kubernetes-version v1.24.4 --profile mini-kubeflow --bootstrapper=kubeadm

# optional args: --extra-config=apiserver.service-account-issuer=api --extra-config=apiserver.service-account-signing-key-file=/var/lib/minikube/certs/apiserver.key --extra-config=apiserver.service-account-api-audiences=api
```
