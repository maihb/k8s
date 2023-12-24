# 安装 k8s

# 最简单的 minikube

```sh
sudo apt install rpm
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.aarch64.rpm
sudo rpm -Uvh minikube-latest.aarch64.rpm
minikube start #failed 
# Exiting due to RSRC_INSUFFICIENT_CORES:  has less than 2 CPUs available, but Kubernetes requires at least 2 to be available

#卸载
sudo rpm -e minikube
```

# 多实例版本  kubeadm

```sh
apt-get update && apt-get install -y apt-transport-https curl
apt-get install -y kubelet kubeadm kubectl --allow-unauthenticated
```