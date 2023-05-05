# Kubeflow Setup

- [Kubeflow Setup](#kubeflow-setup)
  - [Install Kubeflow with raw manifests](#install-kubeflow-with-raw-manifests)


## Install Kubeflow with raw manifests

```bash
git clone https://github.com/kubeflow/manifests.git

cd manifests

# IMPORTANT: Select one of the stable tag from git

while ! kustomize build example | awk '!/well-defined/' | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done

kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
```