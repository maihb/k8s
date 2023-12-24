# 安装 k8s  参考文档： https://todoit.tech/k8s/install/k8s.html

## 0.0 最简单的 minikube

```sh
sudo apt install rpm
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.aarch64.rpm
sudo rpm -Uvh minikube-latest.aarch64.rpm
minikube start #failed 
# Exiting due to RSRC_INSUFFICIENT_CORES:  has less than 2 CPUs available, but Kubernetes requires at least 2 to be available

#卸载
sudo rpm -e minikube
```

## 0.1 多实例版本  kubeadm

```sh
apt-get update && apt-get install -y apt-transport-https curl

# 设置 kubeadm 软件源：
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# 安装 kubelet、kubeadm 和 kubectl，并锁定其版本：
apt update
apt-get install -y kubelet=1.23.15-00 kubeadm=1.23.15-00 kubectl=1.23.15-00
apt-mark hold kubelet kubeadm kubectl
```

## 1.1 禁用swap分区
```sh
#禁用swap分区
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 1.2 依据官方文档允许 iptables 检查桥接流量，作如下设置：

```sh
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

## 2.1 安装 Master 节点
```sh
kubeadm init \
 --kubernetes-version v1.23.15 \
 --pod-network-cidr=10.244.0.0/16 \
 --apiserver-advertise-address=10.1.1.152 \
 --ignore-preflight-errors=NumCPU    
# apiserver-advertise-address 值为 本机 IP ,最后打印
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.1.152:6443 --token 3fohln.wdqxcw08es87gbk7 \
        --discovery-token-ca-cert-hash sha256:af66ecd3ec8a018596f9caff46ee731348bf87bd900b8240a4ecb3b0c856907f 
```