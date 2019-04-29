# K8S 集群搭建

该文档旨在记录搭建 K8S 1.14 集群流程。

基础环境：

- 云厂商虚拟云主机
- Centos 7.2 操作系统 
- root 权限用户

------

### 0. 每台云主机预操作

#### 0.1 更新 YUM 
```
yum update -y
```

#### 0.2 安装 Docker CE

```
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

# 配置 Docker 仓库
sudo yum-config-manager \
	--add-repo \
	https://download.docker.com/linux/centos/docker-ce.repo

# 安装 Docker
sudo yum install docker-ce docker-ce-cli containerd.io -y

# 开启 Docker 
systemctl start docker
```

### 1. 配置 Master 节点

#### 1.0 拉取相关 Docker Image

由于国内墙的问题，需要先从一个仓库拉取镜像，再 tag 为 k8s.gcr.io 地址的镜像

```shell
# 拉取相关镜像
docker pull mirrorgooglecontainers/kube-apiserver:v1.14.1 
docker pull mirrorgooglecontainers/kube-controller-manager:v1.14.1
docker pull mirrorgooglecontainers/kube-scheduler:v1.14.1
docker pull mirrorgooglecontainers/kube-proxy:v1.14.1
docker pull mirrorgooglecontainers/etcd:3.3.10
docker pull coredns/coredns:1.3.1
docker pull mirrorgooglecontainers/pause:3.1

# 重新 tag
docker tag mirrorgooglecontainers/kube-apiserver:v1.14.1 k8s.gcr.io/kube-apiserver:v1.14.1
docker tag mirrorgooglecontainers/kube-controller-manager:v1.14.1 k8s.gcr.io/kube-controller-manager:v1.14.1
docker tag mirrorgooglecontainers/kube-scheduler:v1.14.1 k8s.gcr.io/kube-scheduler:v1.14.1
docker tag mirrorgooglecontainers/kube-proxy:v1.14.1 k8s.gcr.io/kube-proxy:v1.14.1
docker tag mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
```

#### 1.1 安装 kubeadm
由于国内墙的问题，我们不能直接安装 kubeadm ，需要使用阿里云的镜像地址

```
# 修改 yum 配置

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装 kubeadm

yum  install kubeadm  -y
```

#### 1.2 关闭 SELinux & swap
```
# 关闭 SELinux

setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

# 关闭 swap

swapoff -a
```

#### 1.3 设置 docker & kubelet 服务均为开机自启动

```
systemctl enable docker.service
systemctl enable kubelet.service
```

#### 1.4 允许 iptables 对网桥转发的包做过滤

```
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```


#### 1.5 Master Node 初始化

```
kubeadm init --pod-network-cidr=192.168.0.0/16
```

初始化日志中会出现 `kubectl` 的配置命令如下：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
将这几条命令 Copy 并执行。

日志中还将会出现 worker 节点加入集群的命令，示例如下：

```
kubeadm join 10.23.4.232:6443 --token 12uo59.y8nl1hyo29sw7c7s \
    --discovery-token-ca-cert-hash sha256:3f0419eb6fb69e4f57fa65a7d913fd5ceac3e51a590b3f6c45313febfda3ea10
```
改命令将会在 Worker 节点加入集群时在 Worker 节点执行。

#### 1.6 Deploy a pod network to the cluster （这里选择 Calico）

```
kubectl apply -f \
https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

#### 1.7 查看所有 POD 

```
[root@10-23-4-232 docker]#  kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5cbcccc885-xg87t   1/1     Running   0          8m50s
kube-system   calico-node-b7gkt                          1/1     Running   0          8m50s
kube-system   coredns-fb8b8dccf-8gphv                    1/1     Running   0          10m
kube-system   coredns-fb8b8dccf-drlbq                    1/1     Running   0          10m
kube-system   etcd-10-23-4-232                           1/1     Running   0          9m50s
kube-system   kube-apiserver-10-23-4-232                 1/1     Running   0          10m
kube-system   kube-controller-manager-10-23-4-232        1/1     Running   0          10m
kube-system   kube-proxy-d78xl                           1/1     Running   0          10m
kube-system   kube-scheduler-10-23-4-232                 1/1     Running   0          10m
```

如此这般，一切正常

----

### 2. 配置 Worker 节点

#### 2.0 修改镜像源，安装 kubeadm

同 Master Node

#### 2.1 关闭 SELinux & swap 

同 Master Node
#### 2.2 允许 iptables 对网桥转发的包做过滤

同 Master Node
#### 2.3 设置 docker & kubelet 服务均为开机自启动

同 Master Node
#### 2.4 拉取相关镜像，并更改为要求的 tag (同 Master Node)

node节点加入master节点需要网络插件，所以需要以下额外镜像

```
sudo docker pull quay.io/calico/node:v3.3.0
sudo docker pull quay.io/calico/cni:v3.3.0
```

#### 2.5 执行 kubeadm join，加入集群

命令 & 参数在 kubeadm init 时候会输出，过程没有报错的话，切到 Master 节点，检查是否成功加入

```
[root@10-23-4-232 docker]# kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
10-23-115-160   Ready    <none>   9m50s   v1.14.1
10-23-4-232     Ready    master   35m     v1.14.1
```

若新的 node 状态也为 Ready 则表示 OK 了

----

### 3. 常见问题

#### 3.0 running with swap on is not supported. Please disable swap
方案：
swapoff -a


#### 3.1 /bridge-nf-call-iptables contents are not set to 1
方案：
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables


#### 3.2 所有节点必加的开启启动策略：

```
systemctl enable docker.service
systemctl enable kubelet.service
```

#### 3.3 kubeadm init 前预拉取 相关镜像：

`kubeadm config images pull`，这个命令可以在 `kubectl init` 前拉取整个过程中需要的镜像。多数情况下会由于墙的原因而超时失败的，但至少可以知道整个过程中需要准备哪些镜像，然后从其他地方拉取再 `docker tag` 改镜像名。
