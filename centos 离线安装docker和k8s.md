# centos 离线安装 docker,contained,k8s

说明：以下内容在 centos7.9 (最小化安装)测试安装，k8s：1.25.3

## 安装数据包准备

### 说明：

- yum 包准备(在联网设备操作)：可通过查阅 kubernetes（k8s）安装 文中的 `yum install ***` 命令在后面添加 `--downloadonly --downloaddir=./下载的文件夹(都在当前目录下操作)`
- Docker 镜像准备：可通过学习 kubernetes（k8s）安装 安装成功后，使用命令 ctr -n=k8s.io image list 查询 k8s 安装成功后，当前使用的 Docker 镜像，使用命令 ctr -n=k8s.io image export `<image-name>`:`<tag>` `<tar-file>`导出 Docker 镜像到磁盘的文件名 Docker 镜像名
  例子：`ctr -n=k8s.io image export nginx:latest /tmp/nginx.tar`
- Docker 镜像导入：`ctr -n=k8s.io image Docker镜像导出到磁盘的文件名 Docker镜像名`

### rpm 包下载

#### - 准备依赖包

```shell
#准备 wget（可忽略）
sudo yum -y install wget --downloadonly --downloaddir=./wget
#准备 ntpdate（可忽略）
sudo yum -y install ntpdate --downloadonly --downloaddir=./ntpdate
#准备 bash-completion（可忽略，推荐）
sudo yum -y install bash-completion --downloadonly --downloaddir=./bash-completion
#准备 Docker、Containerd 安装前的依赖
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 --downloadonly --downloaddir=./docker-before
```

#### - 准备 Docker 安装包

```powershell
#将 Docker 的 YUM 软件源配置文件 docker-ce.repo 下载并保存到 /etc/yum.repos.d/ 目录下
sudo curl https://download.docker.com/linux/centos/docker-ce.repo > /etc/yum.repos.d/docker-ce.repo
#重新生成yum缓存
sudo yum makecache
#sudo yum clean all && yum makecache #清理缓存并重新生成
#下载 Docker、Docker-Compose 和 Containerd 等软件包及其依赖项
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin containerd --downloadonly --downloaddir=./docker
```

#### - 准备 k8s 安装包

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
# 是否开启本仓库
enabled=1
# 是否检查 gpg 签名文件
gpgcheck=0
# 是否检查 gpg 签名文件
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#重新生成yum缓存
sudo yum makecache
#下载Kubernetes 版本 1.25.3 的 kubelet、kubeadm 和 kubectl 软件包
install -y kubelet-1.25.3-0 kubeadm-1.25.3-0 kubectl-1.25.3-0 --disableexcludes=kubernetes --nogpgcheck --downloadonly --downloaddir=./k8s
```

#### 准备 k8s 初始化所需 Docker 镜像包和 containerd 所需的 Docker 镜像包 pause

```
#以下在安装同款K8s和containerd联网设备操作
#查询当前containerd：1.6.10所需pause版本
sudo containerd config default
#查询结果
#registry.k8s.io/pause:3.6
#registry.k8s.io/etcd:3.5.4-0
#registry.k8s.io/coredns/coredns:v1.9.3
#获取k8s：1.25.3初始化时所需的docker镜像
kubeadm config images list
#查询结果
# registry.k8s.io/kube-apiserver:v1.25.3
# registry.k8s.io/kube-controller-manager:v1.25.3
# registry.k8s.io/kube-scheduler:v1.25.3
# registry.k8s.io/kube-proxy:v1.25.3
# registry.k8s.io/pause:3.8
# registry.k8s.io/etcd:3.5.4-0
# registry.k8s.io/coredns/coredns:v1.9.3

```

docker 镜像拉取

```powershell
# 在这里我们使用阿里云Docker镜像来拉取上面的 Docker image
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.25.3
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.25.3
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.25.3
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.25.3
docker pull registry.aliyuncs.com/google_containers/pause:3.8
# containerd 所需
docker pull registry.aliyuncs.com/google_containers/pause:3.6
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.4-0
docker pull registry.aliyuncs.com/google_containers/coredns:v1.9.3

#查看镜像
docker images
# 将上述的 registry.aliyuncs.com 修改为 registry.k8s.io
docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.25.3            registry.k8s.io/kube-apiserver:v1.25.3
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.25.3            registry.k8s.io/kube-scheduler:v1.25.3
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.25.3   registry.k8s.io/kube-controller-manager:v1.25.3
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.25.3                registry.k8s.io/kube-proxy:v1.25.3
docker tag registry.aliyuncs.com/google_containers/pause:3.8                         registry.k8s.io/pause:3.8
# containerd 所需
docker tag registry.aliyuncs.com/google_containers/pause:3.6                         registry.k8s.io/pause:3.6
docker tag registry.aliyuncs.com/google_containers/etcd:3.5.4-0                      registry.k8s.io/etcd:3.5.4-0
# 注意这里的名称为 coredns/coredns:v1.9.3
docker tag registry.aliyuncs.com/google_containers/coredns:v1.9.3                    registry.k8s.io/coredns/coredns:v1.9.3

# 保存镜像到磁盘(当前目录)
docker save -o kube-apiserver-v1.25.3.tar            registry.k8s.io/kube-apiserver:v1.25.3
docker save -o kube-controller-manager-v1.25.3.tar   registry.k8s.io/kube-controller-manager:v1.25.3
docker save -o kube-scheduler-v1.25.3.tar            registry.k8s.io/kube-scheduler:v1.25.3
docker save -o kube-proxy-v1.25.3.tar                registry.k8s.io/kube-proxy:v1.25.3
docker save -o pause-3.8.tar                         registry.k8s.io/pause:3.8
# containerd 所需
docker save -o pause-3.6.tar                         registry.k8s.io/pause:3.6
docker save -o etcd-3.5.4-0.tar                      registry.k8s.io/etcd:3.5.4-0
docker save -o coredns-v1.9.3.tar                    registry.k8s.io/coredns/coredns:v1.9.3
# 将上述镜像复制到已安装 k8s、待初始化 k8s 的系统上
```

#### 准备网络 calico 初始化所需 docker 镜像包

本次使用 calico 3.24.5。其他版本查看对应的 calicao.yaml 文件中 calico/node、calico/cni、calico/kube-controllers 版本，下载对应的 Docker 镜像即可

calico GitCode 加速镜像： [https://gitcode.net/mirrors/projectcalico/calico/-/raw/v3.24.5/manifests/calico.yaml](https://gitcode.net/mirrors/projectcalico/calico/-/raw/v3.24.5/manifests/calico.yaml)

```
docker pull docker.io/calico/node:v3.24.5
docker pull docker.io/calico/cni:v3.24.5
docker pull docker.io/calico/kube-controllers:v3.24.5

docker images

docker save -o node-v3.24.5.tar                    docker.io/calico/node:v3.24.5
docker save -o cni-v3.24.5.tar                     docker.io/calico/cni:v3.24.5
docker save -o kube-controllers-v3.24.5.tar        docker.io/calico/kube-controllers:v3.24.5

```

## 安装

说明：以下安装过程在服务器上传安装数据所在文件夹内进行

```powershell
ls
#bash-completion  calico  calico.yaml  docker  docker-before  init-images  k8s  nginx-1.23.2.tar  ntpdate  vim  wget
```

### 安装 vim

```
cd ./vim
#用于从本地系统安装 RPM 包
yum -y localinstall *.rpm
# yum -y install *.rpm
cd ..
```

其他 wget,ntpdate,bash-completion 根据需求进入相应文件夹安装

### 安装 Docker、Containerd 安装前的依赖

```powershell
cd ./docker-before
yum -y localinstall *.rpm
# yum -y install *.rpm
cd ..
```

### 安装 Docker、Containerd

```powershell
cd ./docker
#安装前卸载旧版本docker详见docker卸载
yum -y localinstall *.rpm
# yum -y install *.rpm
cd ..

# 启动 docker 时，会启动 containerd
# sudo systemctl status containerd.service
sudo systemctl stop containerd.service

sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
sudo containerd config default > $HOME/config.toml
sudo cp $HOME/config.toml /etc/containerd/config.toml

# 由于是离线安装，提前准备了Docker镜像，所以此处不用修改 pause

# https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd
# 确保 /etc/containerd/config.toml 中的 disabled_plugins 内不存在 cri
sudo sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml

sudo systemctl enable --now containerd.service
# sudo systemctl status containerd.service

# sudo systemctl status docker.service
sudo systemctl start docker.service
# sudo systemctl status docker.service
sudo systemctl enable docker.service
sudo systemctl enable docker.socket
sudo systemctl list-unit-files | grep docker

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://hnkfbj7x.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker info

```

查看 docker 状态，并查看 containerd 是否伴随启动

```powershell
sudo systemctl status docker.service
sudo systemctl status containerd.service
```

### 安装 k8s

```shell
# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

cd k8s
yum -y localinstall *.rpm
# yum -y install *.rpm
cd ..

systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl enable kubelet

```

### 导入 k8s 初始化时所需的 Docker 镜像

```powershell
cd init-images

# 注意这里指定了命名空间为 k8s.io
ctr -n=k8s.io image import kube-apiserver-v1.25.3.tar
ctr -n=k8s.io image import kube-controller-manager-v1.25.3.tar
ctr -n=k8s.io image import kube-scheduler-v1.25.3.tar
ctr -n=k8s.io image import kube-proxy-v1.25.3.tar
ctr -n=k8s.io image import pause-3.8.tar
#  containerd 使用
ctr -n=k8s.io image import pause-3.6.tar
ctr -n=k8s.io image import etcd-3.5.4-0.tar
ctr -n=k8s.io image import coredns-v1.9.3.tar

ctr -n=k8s.io images list
ctr i list

cd ..

```

### 将主机名指向本机 IP

**主机名只能包含：字母、数字、-（横杠）、.（点）**

```shell
#获取主机名
hostname
#临时设置主机名
hostname 主机名
#永久设置主机名
sudo echo '主机名' > /etc/hostname
#编辑 hosts
sudo vim /etc/hosts
#末尾添加
#当前机器的IP 	当前机器的主机名
```

### 设置防火墙开放

```powershell
#关闭防火墙(生产环境不推荐)
sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service
```

```shell
# 开放所需端口
# 控制面板
firewall-cmd --zone=public --add-port=6443/tcp --permanent # Kubernetes API server	所有
firewall-cmd --zone=public --add-port=2379/tcp --permanent # etcd server client API	kube-apiserver, etcd
firewall-cmd --zone=public --add-port=2380/tcp --permanent # etcd server client API	kube-apiserver, etcd
firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API	自身, 控制面
firewall-cmd --zone=public --add-port=10259/tcp --permanent # kube-scheduler	自身
firewall-cmd --zone=public --add-port=10257/tcp --permanent # kube-controller-manager	自身
firewall-cmd --zone=trusted --add-source=192.168.80.60 --permanent # 信任集群中各个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.80.16 --permanent # 信任集群中各个节点的IP
firewall-cmd --add-masquerade --permanent # 端口转发
firewall-cmd --reload
firewall-cmd --list-all
firewall-cmd --list-all --zone=trusted

# 工作节点
firewall-cmd --zone=public --add-port=10250/tcp --permanent # Kubelet API	自身, 控制面
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent # NodePort Services†	所有
firewall-cmd --zone=trusted --add-source=192.168.80.60 --permanent # 信任集群中各个节点的IP
firewall-cmd --zone=trusted --add-source=192.168.80.16 --permanent # 信任集群中各个节点的IP
firewall-cmd --add-masquerade --permanent # 端口转发
firewall-cmd --reload
firewall-cmd --list-all
firewall-cmd --list-all --zone=trusted
```

### 关闭交换空间

```shell
sudo swapoff -a
sudo sed -i 's/.*swap.*/#&/' /etc/fstab
```

### k8s 初始化

```shell
# 由于导入的 Docker 镜像已经修改为原始的名称，故此处初始化无需增加 --image-repository=registry.aliyuncs.com/google_containers 本机ip192.168.80.60
kubeadm init --apiserver-advertise-address=192.168.80.60
# 指定集群的IP
# kubeadm init --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.80.60

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info

# 初始化失败后，可进行重置，重置命令：kubeadm reset

# 执行成功后，会出现类似下列内容：
# kubeadm join 192.168.80.60:6443 --token f9lvrz.59mykzssqw6vjh32 \
# --discovery-token-ca-cert-hash sha256:4e23156e2f71c5df52dfd2b9b198cce5db27c47707564684ea74986836900107
# 或者
#Kubernetes control plane is running at https://172.17.93.185:6443
#CoreDNS is running at https://172.17.93.185:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
#To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

```

### 网络初始化

```
cd ./calico
ctr -n=k8s.io image import node-v3.24.5.tar
ctr -n=k8s.io image import cni-v3.24.5.tar
ctr -n=k8s.io image import kube-controllers-v3.24.5.tar
cd ..
```

增加 dns

```shell
vim /etc/resolv.conf
#nameserver 192.168.10.1
```

修改 calico.yaml 文件

```
# 在 - name: CLUSTER_TYPE 下方添加如下内容
- name: CLUSTER_TYPE
  value: "k8s,bgp"
  # 下方为新增内容
- name: IP_AUTODETECTION_METHOD
  value: "interface=网卡名称"
```

应用

```shell
kubectl apply -f calico.yaml
```

### 查看集群

```shell
kubectl get pods --all-namespaces -o wide
kubectl get nodes -o wide
```

## 附录

### 去污

控制面板去污用于单节点集群

### 创建实例

### 其他命令

### docker 卸载

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

docker 服务重启

```shell
#查看docker状态
service docker status
#停止docker服务
systemctl stop docker
#启动docker服务
systemctl start docker
#重启docker服务
systemctl restart docker / service docker restart
```

上传 docker 所需 rpm 包（在 docker 文件夹）
