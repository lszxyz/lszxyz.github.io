Title: debian deploy kubenetes 
Date: 2024-6-12 20:00
Modified: 2024-6-12 20:00
Category: kubenetes
Tags: pelican, publishing
Slug: my-super-post
Authors: zero

# kubenetes碎碎念 debian12 install  ha kubenetes with kube-vip 

## 基本规划:

|    ip    | hostname |
| -------- | ------- |
| 192.168.2.202 | kubenode2 |
| 192.168.2.203 | kubenode3 |
| 192.168.2.204 | kubenode4 | 
| 192.168.2.205 | kubevip.cluster.local | 

## 系统版本信息
```sh
root@kube-node-2:~# cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

## 更改系统配置
```sh
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

swapoff -a
```

## 安装 containerd、runc、kubeadm、kubectl、kubelet、
```sh
export CONTAINERD_VERSION=1.7.18
export RUNC_VERSION=v1.1.12
export CNI_PLUGINS_VERSION=v1.5.0
export NERDCTL_VERSION=1.7.6
export K9S_VERSION=0.32.5

# install nerdctl
wget https://github.com/containerd/nerdctl/releases/download/v${NERDCTL_VERSION}/nerdctl-"${NERDCTL_VERSION}"-linux-amd64.tar.gz
tar -zxvf nerdctl-"${NERDCTL_VERSION}"-linux-amd64.tar.gz
mv nerdctl /usr/local/bin/

# install containerd
wget https://github.com/containerd/containerd/releases/download/v"${CONTAINERD_VERSION}"/containerd-"${CONTAINERD_VERSION}"-linux-amd64.tar.gz
tar xvf containerd-"${CONTAINERD_VERSION}"-linux-amd64.tar.gz
mv ./bin/* /usr/local/bin/

mkdir -p /usr/local/lib/systemd/system/
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -O "/usr/local/lib/systemd/system/containerd.service"
systemctl daemon-reload
systemctl enable --now containerd
systemctl status containerd

# install runc
wget https://github.com/opencontainers/runc/releases/download/"${RUNC_VERSION}"/runc.amd64 -O "/usr/local/sbin/runc"
chmod +x /usr/local/sbin/runc

# cni plugin
wget https://github.com/containernetworking/plugins/releases/download/"${CNI_PLUGINS_VERSION}"/cni-plugins-linux-amd64-"${CNI_PLUGINS_VERSION}".tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-"${CNI_PLUGINS_VERSION}".tgz

# add containerd config 
mkdir -p /etc/containerd
touch /etc/containerd/config.toml

# install kubeadm kubectl kubelet 
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gpg
mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' |  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# install k9s 
wget https://github.com/derailed/k9s/releases/download/v"${K9S_VERSION}"/k9s_Linux_amd64.tar.gz
tar -zxvf k9s_Linux_amd64.tar.gz
mv k9s /usr/local/bin
```

## 初始化集群
```sh
# 打印默认配置文件
kubeadm config print init-defaults > kubeadm-config.yaml
```
这里配置根据自己情况修改、具体参考如下连接:

  - [kubernetes-reliability.md](https://github.com/kubernetes-sigs/kubespray/blob/b77f2075128397cda7165e31f930e0f2b6ffe535/docs/kubernetes-reliability.md)

  - [ha-considerations.md](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md)

```sh
# 提前下载镜像
kubeadm config images list  --config=kubeadm-config.yaml
ctr  --namespace k8s.io image pull registry.k8s.io/kube-apiserver:v1.30.0
ctr  --namespace k8s.io image pull registry.k8s.io/kube-controller-manager:v1.30.0
ctr  --namespace k8s.io image pull registry.k8s.io/kube-scheduler:v1.30.0
ctr  --namespace k8s.io image pull registry.k8s.io/kube-proxy:v1.30.0
ctr  --namespace k8s.io image pull registry.k8s.io/coredns/coredns:v1.11.1
ctr  --namespace k8s.io image pull registry.k8s.io/pause:3.9
ctr  --namespace k8s.io image pull registry.k8s.io/etcd:3.5.12-0

export VIP=192.168.2.205
export INTERFACE=enp1s0
apt-get install jq 
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml

kubeadm init --config=kubeadm-config.yaml --upload-certs --v=5
# 这里会报错、相关issue参考  https://github.com/kube-vip/kube-vip/issues/684、更改下kube-vip配置重新初始化即可、reset之后上面kube-vip的要重新跑一下
kubeadm reset -f
sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' \
          /etc/kubernetes/manifests/kube-vip.yaml
kubeadm init --config=kubeadm-config.yaml --upload-certs --v=5
```

## 其他节点加入
```sh
scp  /etc/kubernetes/manifests/kube-vip.yaml root@kubenode3:/etc/kubernetes/manifests
kubeadm join 192.168.2.205:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx  --control-plane --certificate-key xxx --v=5
kubectl get node -o wide 
```

