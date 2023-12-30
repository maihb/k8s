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

## 0.1 多实例版本  kubeadm ubuntu

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

## 0.2 多实例版本  kubeadm centos7
```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet-1.23.15 kubeadm-1.23.15 kubectl-1.23.15 --disableexcludes=kubernetes
#yum remove  kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet

#关闭防火墙
firewall-cmd --state
systemctl stop firewalld.service
systemctl disable firewalld.service

#安装缓存工具
yum -y install systemd-resolved
systemctl start systemd-resolved && systemctl status systemd-resolved
systemctl enable systemd-resolved
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

kubeadm init --apiserver-advertise-address=10.10.0.3  --service-cidr=20.20.0.0/16 --pod-network-cidr=10.244.0.0/16 --kubernetes-version v1.21.1

# 云服务器多实例需要在路由表添加下面几个 IP 路由设置，确保互通。
# apiserver-advertise-address 值为 本机 IP ,最后打印
# 下面 2 个 cidr，如果是多实例 ip， 需要手动确认路由表是否可通。否则无法访问 services
kubeadm init \
 --kubernetes-version v1.23.15 \
 --pod-network-cidr=10.10.0.0/16 \
 --service-cidr=20.20.0.0/16 \
 --apiserver-advertise-address=10.1.1.152 \
 --ignore-preflight-errors=NumCPU    

# 输出：
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.1.152:6443 --token 3fohln.wdqxcw08es87gbk7 \
        --discovery-token-ca-cert-hash sha256:af66ecd3ec8a018596f9caff46ee731348bf87bd900b8240a4ecb3b0c856907f 

#token 具有时效性，如果 token 过期，可以在 Master 节点上重新生成
kubeadm token create --print-join-command

#重置： kubeadm reset

#非root用户使用kubectl --切换回非 root 用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

## 如果遇到 join 失败：
先测试 telnet 10.1.1.152 6443 失败，可能是防火墙有问题，清掉规则重新来   
root@v-db:~# iptables -F   
root@v-db:~# service docker restart   

## It seems like the kubelet isn't running or healthy.
journalctl -xefu kubelet   
 查询系统日志可知：“Failed to run kubelet” err="failed to run Kubelet: misconfiguration: kubelet driver: “cgroupfs” is different from docker cgroup driver: “systemd”    
解决办法：  docker配置添加 “exec-opts”: [“native.cgroupdriver=systemd”]   
vim /etc/docker/daemon.json  # 添加：  
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
重启 docker  和  kubelet   
service docker restart   
service kubelet restart   

最后检测：
ubuntu@v-db:~$ kubectl get node
NAME    STATUS   ROLES                  AGE    VERSION
c2      Ready    <none>                 8m9s   v1.23.15
v-db    Ready    control-plane,master   140m   v1.23.15
vnic1   Ready    <none>                 99m    v1.23.15

## 安装网络插件
copy from git地址：https://github.com/coreos/flannel#flannel  
kubectl apply -f https://raw.githubusercontent.com/maihb/k8s/master/kube-flannel.yml


## 删除 Worker 节点
如果希望删除 Worker 节点，可以在 Master 节点上执行如下命令  
```sh
kubectl get nodes  
kubectl drain v2 --delete-local-data --force --ignore-daemonsets   #node NAME = v2
kubectl delete node v2 
```
## 自动补全
```sh
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
sudo chmod a+r /etc/bash_completion.d/kubectl
echo 'alias k=kubectl' >>/etc/profile
echo 'complete -o default -F __start_kubectl k' >>/etc/profile
```