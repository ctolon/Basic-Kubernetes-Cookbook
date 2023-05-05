# Kubernetes Commands

## Fundamental Commands

```bash
kubectl get pods -A
kubectl get pods -A --watch

kubectl get pods --all-namespaces

kubectl get services --all-namespace

kubectl get componentstatuses

kubectl get nodes

kubectl get certificate

sudo kubeadm reset --v 5 --cri-socket=unix:///var/run/cri-dockerd.sock
sudo kubeadm reset --v 5 --cri-socket=unix:///var/run/containerd.sock

iptables -I INPUT -p TCP --dport 443 -j ACCEPT

kubectl config view

kubectl cluter-info

kubectl create configmap kube-dns --from-file=/etc/kubernetes/kube-dns-config.yaml --namespace=kube-system
kubectl get configmaps -n kube-system
kubectl get configmap -n kube-system kube-dns -o jsonpath='{.data.ca\.crt}' > ca.crt

kubectl logs --namespace=kube-system -l k8s-app=kube-dns
kubectl get pods -n kube-system -l k8s-app=kube-dns

journalctl -u kube-apiserver -f

kubectl get nodes -o wide

```

## Purge Calico Configs

Steps to remove old calico configs from kubernetes without kubeadm reset:

* clear ip route
* remove all calico links in all nodes
* remove ipipmodule
* remove calico configs
* restart


```sh
ip route flush proto bird
ip link list | grep cali | awk '{print $2}' | cut -c 1-15 | xargs -I {} ip link delete {}
modprobe -r ipip
rm /etc/cni/net.d/10-calico.conflist && rm /etc/cni/net.d/calico-kubeconfig
kubelet service kubelet restart
```



