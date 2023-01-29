# 1. 使用kubeadm部署一个分布式的Kubernetes集群
## 1.1 设定时钟同步 
* 若节点可直接访问互联网，安装chrony程序包后，可直接启动chronyd系统服务，并设定其随系统引导而启动。随后，chronyd服务即能够从默认的时间服务器同步时间。
* 所有节点都需要设置，这里只展示一台节点的安装命令
```bash
## 安装chrony程序包
root@k8s-master1:~# apt install chrony
## 启动chrony程序包
root@k8s-master1:~# systemctl start chrony.service
## 将chrony程序加入开启自启动
root@k8s-master1:~# systemctl enable chrony.service
## 查看服务装填
root@k8s-master1:~# systemctl status chrony.service
```
## 1.2 主机名称解析
* 测试环境使用hosts文件进行各节点名称解析，文件内容如下所示。
```bash
11.0.1.201      k8s-master1.ygc.cn      k8s-master1
11.0.1.101      k8s-node1.ygc.cn        k8s-node1
11.0.1.102      k8s-node2.ygc.cn        k8s-node2
11.0.1.103      k8s-node3.ygc.cn        k8s-node3
```
## 1.3 禁用Swap设备
* 部署集群时，kubeadm默认会预先检查当前主机是否禁用了Swap设备，并在未禁用时强制终止部署过程。因此，在主机内存资源充裕的条件下，需要禁用所有的Swap设备，否则，就需要在后文的kubeadm init及kubeadm join命令执行时额外使用相关的选项忽略检查错误。
* 所有节点都需要设置，这里只展示一台节点的操作命令
```bash
## 关闭Swap设备，需要分两步完成。首先是关闭当前已启用的所有Swap设备
root@k8s-master1:~# swapoff -a
## 注释用于挂载Swap设备的所有行
root@k8s-master1:~# vi /etc/fstab
#/swap.img      none    swap    sw      0       0
## 另外，在Ubuntu 2004及之后版本的系统上，若要彻底禁用Swap，可以需要类似如下命令进一步完成。
root@k8s-master1:~# systemctl --type swap --all
  UNIT LOAD ACTIVE SUB DESCRIPTION
0 loaded units listed.
To show all installed unit files use 'systemctl list-unit-files'.
## 将上面命令列出的每个设备，使用systemctl mask命令加以禁用
root@k8s-master1:~# systemctl mask SWAP_DEV
```
## 1.4 禁用默认的防火墙服务
* Ubuntu和Debian等Linux发行版默认使用ufw（Uncomplicated FireWall）作为前端来简化 iptables的使用，处于启用状态时，它默认会生成一些规则以加强系统安全。出于降低配置复杂度之目的，本文选择直接将其禁用。
```bash
## 关闭防火墙
root@k8s-master1:~# ufw disable
## 查看防火墙状态
root@k8s-master1:~# ufw status
```
## 1.5 安装程序包
* 所有节点都需要安装，这里只展示一台节点的操作命令
```bash
## 生成docker-ce相关程序包的仓库
root@k8s-master1:~# apt -y install apt-transport-https ca-certificates curl software-properties-common
root@k8s-master1:~# curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
root@k8s-master1:~# add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
root@k8s-master1:~# apt update
## 安装docker-ce
root@k8s-master1:~# apt install docker-ce
## 编辑docker的配置文件
root@k8s-master1:~# vi /etc/docker/daemon.json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "200m"
  },
  "storage-driver": "overlay2"
}
## 重新加载服务的配置文件
root@k8s-master1:~# systemctl daemon-reload
## 启动docker服务
root@k8s-master1:~# systemctl start docker.service
## 查看docker服务
root@k8s-master1:~# systemctl status docker.service
## 将docker服务加入开机启动项
root@k8s-master1:~# systemctl enable docker.service
```
## 1.6 安装cri-dockerd
* Kubernetes自v1.24移除了对docker-shim的支持，而Docker Engine默认又不支持CRI规范，因而二者将无法直接完成整合。为此，Mirantis和Docker联合创建了cri-dockerd项目，用于为Docker Engine提供一个能够支持到CRI规范的垫片，从而能够让Kubernetes基于CRI控制Docker 。
  * 项目地址：https://github.com/Mirantis/cri-dockerd
* cri-dockerd项目提供了预制的二进制格式的程序包，用户按需下载相应的系统和对应平台的版本即可完成安装
```bash
## 下载安装包，节点使用系统Ubuntu 2204 64bits
root@k8s-master1:~# curl -LO https://ghproxy.com/https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.0/cri-dockerd_0.3.0.3-0.ubuntu-jammy_amd64.deb
root@k8s-master1:~# dpkg -i cri-dockerd_0.3.0.3-0.ubuntu-jammy_amd64.deb
## 查看服务状态
root@k8s-master1:~# systemctl status cri-docker.service
```
## 1.7 安装kubelet、kubeadm和kubectl
* 在各主机上生成kubelet和kubeadm等相关程序包的仓库，使用阿里云的镜像服务
```bash
root@k8s-master1:~# apt update && apt install -y apt-transport-https curl
root@k8s-master1:~# curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
root@k8s-master1:~# cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
root@k8s-master1:~# apt update
## 安装kubelet、kubeadm和kubectl等程序包，并将其设置为随系统启动而自动引导
root@k8s-master1:~# apt install -y kubelet kubeadm kubectl
root@k8s-master1:~# systemctl enable kubelet
```
## 1.8 整合kubelet和cri-dockerd
* 仅支持CRI规范的kubelet需要经由遵循该规范的cri-dockerd完成与docker-ce的整合
### 1.8.1 配置cri-dockerd
  * 配置cri-dockerd，确保其能够正确加载到CNI插件。编辑/usr/lib/systemd/system/cri-docker.service文件，确保其[Service]配置段中的ExecStart的值类似如下内容
```bash
root@k8s-master1:~# vi /usr/lib/systemd/system/cri-docker.service
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --cni-bin-dir=/opt/cni/bin --cni-cache-dir=/var/lib/cni/cache --cni-conf-dir=/etc/cni/net.d
```
* 需要添加的各配置参数（各参数的值要与系统部署的CNI插件的实际路径相对应）：

  * --network-plugin：指定网络插件规范的类型，这里要使用CNI；
  * --cni-bin-dir：指定CNI插件二进制程序文件的搜索目录；
  * --cni-cache-dir：CNI插件使用的缓存目录；
  * --cni-conf-dir：CNI插件加载配置文件的目录；
* 配置完成后，重载并重启cri-docker.service服务。
```bash
root@k8s-master1:~# systemctl daemon-reload && systemctl restart cri-docker.service
```
### 1.8.2 配置kubelet
* 配置kubelet，为其指定cri-dockerd在本地打开的Unix Sock文件的路径，该路径一般默认为“/run/cri-dockerd.sock“。编辑文件/etc/sysconfig/kubelet，为其添加 如下指定参数
```bash
root@k8s-master1:~# mkdir /etc/sysconfig
root@k8s-master1:~# vi /etc/sysconfig/kubelet
KUBELET_KUBEADM_ARGS="--container-runtime=remote --container-runtime-endpoint=/run/cri-dockerd.sock"
root@k8s-master1:~# systemctl restart kubelet
```
## 1.9 初始化节点
* 该步骤开始尝试构建Kubernetes集群的master节点，配置完成后，各worker节点直接加入到集群中的即可。需要特别说明的是，由kubeadm部署的Kubernetes集群上，集群核心组件kube-apiserver、kube-controller-manager、kube-scheduler和etcd等均会以静态Pod的形式运行，它们所依赖的镜像文件默认来自于registry.k8s.io这一Registry服务之上。但我们无法直接访问该服务，常用的解决办法有如下两种，本示例将选择使用更易于使用的前一种方式。
  * 使用能够到达该服务的代理服务；
  * 使用国内的镜像服务器上的服务，例如registry.aliyuncs.com/google_containers等
### 1.9.1 初始化master节点（在master1上完成如下操作） 
* 在运行初始化命令之前先运行如下命令单独获取相关的镜像文件，而后再运行后面的kubeadm init命令，以便于观察到镜像文件的下载过程。
```bash
## 打印 kubeadm 要使用的镜像列表,使用阿里云镜像加速服务。
root@k8s-master1:~# kubeadm config images list --image-repository=registry.aliyuncs.com/google_containers

registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.1
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.26.1
registry.aliyuncs.com/google_containers/kube-scheduler:v1.26.1
registry.aliyuncs.com/google_containers/kube-proxy:v1.26.1
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.6-0
registry.aliyuncs.com/google_containers/coredns:v1.9.3
## 手动拉取kubeadm 镜像
root@k8s-master1:~# kubeadm config images pull --cri-socket unix:///run/cri-dockerd.sock --image-repository=registry.aliyuncs.com/google_containers
## k8s-master1节点的初始化
root@k8s-master1:~# kubeadm init --control-plane-endpoint="kubeapi.ygc.cn" --kubernetes-version=v1.26.0 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --cri-socket unix:///run/cri-dockerd.sock --upload-certs  --image-repository=registry.aliyuncs.com/google_containers

#如果初始化报如下错误：
Error getting node" err="node \"k8s-master01\" not found

#1、在cri-docker.service文件指定下pause版本：
root@k8s-master1:~# vim /usr/lib/systemd/system/cri-docker.service
ExecStart=/usr/bin/cri-dockerd --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.8 --container-runtime-endpoint fd:// --network-plugin=cni --cni-bin-dir=/opt/cni/bin --cni-cache-dir=/var/lib/cni/cache --cni-conf-dir=/etc/cni/net.d
#2、重启服务：
root@k8s-master1:~# systemctl daemon-reload
root@k8s-master1:~# systemctl restart cri-docker.service

#3、重置集群：
root@k8s-master1:~# kubeadm reset --cri-socket unix:///run/cri-dockerd.sock && rm -rf /etc/kubernetes/ /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni /etc/cni/net.d
```
* 命令中的各选项简单说明如下：
  * --image-repository：指定要使用的镜像仓库，默认为registry.k8s.io；
  * --kubernetes-version：kubernetes程序组件的版本号，它必须要与安装的kubelet程序包的版本号相同；
  * --control-plane-endpoint：控制平面的固定访问端点，可以是IP地址或DNS名称，会被用于集群管理员及集群组件的kubeconfig配置文件的API Server的访问地址；单控制平面部署时可以不使用该选项；
  * --pod-network-cidr：Pod网络的地址范围，其值为CIDR格式的网络地址，通常，Flannel网络插件的默认为10.244.0.0/16，Project Calico插件的默认值为192.168.0.0/16；
  * --service-cidr：Service的网络地址范围，其值为CIDR格式的网络地址，默认为10.96.0.0/12；通常，仅Flannel一类的网络插件需要手动指定该地址；
  * --apiserver-advertise-address：apiserver通告给其他组件的IP地址，一般应该为Master节点的用于集群内部通信的IP地址，0.0.0.0表示节点上所有可用地址；
  * --token-ttl：共享令牌（token）的过期时长，默认为24小时，0表示永不过期；为防止不安全存储等原因导致的令牌泄露危及集群安全，建议为其设定过期时长。未设定该选项时，在token过期后，若期望再向集群中加入其它节点，可以使用如下命令重新创建token，并生成节点加入命令。
### 1.9.2 初始化完成后的操作步骤
* 命令运行结束后,初始化成功输出信息
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kubeapi.ygc.cn:6443 --token 9idhhl.7kluv2mlg1phcthy \
	--discovery-token-ca-cert-hash sha256:bb8c5a9efb2e908221d04337234eff8d4b7ef1a9526d1009e25d1f7bb964b4a2 \
	--control-plane --certificate-key 0cc4f9b809b0e5e01465403e49f4346221a4fa0fd841ddf6c47f2eeec796b246

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubeapi.ygc.cn:6443 --token 9idhhl.7kluv2mlg1phcthy \
	--discovery-token-ca-cert-hash sha256:bb8c5a9efb2e908221d04337234eff8d4b7ef1a9526d1009e25d1f7bb964b4a2
```
### 1.9.3 设定kubectl
* kubectl是kube-apiserver的命令行客户端程序，实现了除系统部署之外的几乎全部的管理操作，是kubernetes管理员使用最多的命令之一。kubectl需经由API server认证及授权后方能执行相应的管理操作，kubeadm部署的集群为其生成了一个具有管理员权限的认证配置文件/etc/kubernetes/admin.conf，它可由kubectl通过默认的“$HOME/.kube/config”的路径进行加载。当然，用户也可在kubectl命令上使用--kubeconfig选项指定一个别的位置。

* 下面复制认证为Kubernetes系统管理员的配置文件至目标用户（例如当前用户root）的家目录下
```bash
root@k8s-master1:~# mkdir ~/.kube
root@k8s-master1:~# cp /etc/kubernetes/admin.conf  ~/.kube/config
root@k8s-master1:~# kubectl get node
NAME          STATUS     ROLES           AGE     VERSION
k8s-master1   NotReady   control-plane   4m54s   v1.26.1
```
## 1.10 部署网络插件
* Kubernetes系统上Pod网络的实现依赖于第三方插件进行，这类插件有近数十种之多，较为著名的有flannel、calico、canal和kube-router等，简单易用的实现是为CoreOS提供的flannel项目。下面的命令用于在线部署flannel至Kubernetes系统之上：
* 首先，下载适配系统及硬件平台环境的flanneld至每个节点，并放置于/opt/bin/目录下。我们这里选用flanneld-amd64，目前最新的版本为v0.20.2，因而，我们需要在集群的每个节点上执行如下命令：
```bash
root@k8s-master1:~# mkdir /opt/bin/
root@k8s-master1:~# curl -L https://github.com/flannel-io/flannel/releases/download/v0.20.2/flanneld-amd64  -o /opt/bin/flanneld
root@k8s-master1:~#  chmod +x /opt/bin/flanneld
```
* 部署kube-flannel
```bash
## 部署kube-flannel
root@k8s-master1:~# kubectl apply -f https://ghproxy.com/https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
## 查看部署状态
root@k8s-master1:~# kubectl get pods -n kube-flannel
NAME                    READY   STATUS    RESTARTS   AGE
kube-flannel-ds-8k6tc   1/1     Running   0          23s
```
## 1.11 验证master节点已经就绪
```bash
root@k8s-master1:~# kubectl get node
NAME          STATUS   ROLES           AGE   VERSION
k8s-master1   Ready    control-plane   14m   v1.26.1
```
## 1.12 添加节点到集群中
* 步骤需要分别在k8s-node1、k8s-node2和k8s-node3上各自完成
```bash
root@k8s-node1:~#kubeadm join kubeapi.ygc.cn:6443 --token 9idhhl.7kluv2mlg1phcthy --discovery-token-ca-cert-hash sha256:bb8c5a9efb2e908221d04337234eff8d4b7ef1a9526d1009e25d1f7bb964b4a2  --cri-socket unix:///run/cri-dockerd.sock
```
## 1.13 验证节点添加结果
```bash
root@k8s-master1:~# kubectl get node
NAME          STATUS   ROLES           AGE     VERSION
k8s-master1   Ready    control-plane   25m     v1.26.1
k8s-node1     Ready    <none>          8m22s   v1.26.1
k8s-node2     Ready    <none>          8m6s    v1.26.1
k8s-node3     Ready    <none>          8m      v1.26.1
```
## 1.14测试应用编排及服务访问
* 到此为止，一个master，并附带有三个worker的kubernetes集群基础设施已经部署完成，用户随后即可测试其核心功能。例如，下面的命令可将demoapp以Pod的形式编排运行于集群之上，并通过在集群外部进行访问：
```bash
## 创建一个deployment应用
root@k8s-master1:~# kubectl create deployment demoapp --image=ikubernetes/demoapp:v1.0 --replicas=3
## 查看pod
root@k8s-master1:~# kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
demoapp-75f59c894-f5b8b   1/1     Running   0          36s
demoapp-75f59c894-lvpp6   1/1     Running   0          36s
demoapp-75f59c894-vnbjx   1/1     Running   0          36s
## 创建service
root@k8s-master1:~# kubectl create service nodeport demoapp --tcp=80:80
## 查看service的信息
root@k8s-master1:~# kubectl get svc -l app=demoapp
NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
demoapp   NodePort   10.102.57.1   <none>        80:31832/TCP   40s
## 运行一个临时客户端pod
root@k8s-master1:~#  kubectl run client-$RANDOM --image=ikubernetes/admin-box:v1.2 --rm --restart=Never -it --command -- /bin/bash
root@client-27662 /# while true; do curl demoapp.default.svc; sleep 1; done
## 命令会持续返回类似如下的结果，这可以证明CoreDNS名称解析功能，以及Service的服务发现及负载均衡功能均已经正常工作。
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-f5b8b, ServerIP: 10.244.1.2!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-lvpp6, ServerIP: 10.244.3.2!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-lvpp6, ServerIP: 10.244.3.2!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-f5b8b, ServerIP: 10.244.1.2!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-lvpp6, ServerIP: 10.244.3.2!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.2.3, ServerName: demoapp-75f59c894-vnbjx, ServerIP: 10.244.2.2!
```
# 2. 扩展作业：使用kubeasz部署一个分布式的Kubernetes集群
* 集群所需节点配置如下

| 角色       | 服务器                                                 | 描述                                |
|----------|-----------------------------------------------------|-----------------------------------|
| 部署节点     | 192.168.50.201                                      | 作为宿主机通过kubeasz容器运行ansible/ezctl命令 |
| etcd节点   | 192.168.50.101<br/>192.168.50.102<br/>192.168.50.103 | 注意etcd集群需要1,3,5,…奇数个节点，本实战安装1个节点  |
| master节点 | 192.168.50.201<br/>192.168.50.202                        | 高可用集群至少2个master节点，测试安装1个节点        |
| node节点   | 192.168.50.101<br/>192.168.50.102<br/>192.168.50.103 | 运行应用负载的节点，节点数任意，测试安装3个节点          |
## 2.1 准备环境
### 2.1.1 准备脚本、二进制文件和镜像文件
```bash
## 安装依赖包
root@k8s-master1:~# apt install -y libseccomp-dev
## 设置kubeasz版本
root@k8s-master1:~# export release=3.5.0
## 下载kubeasz
root@k8s-master1:~# wget https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
root@k8s-master1:~# chmod +x ./ezdown
## 使用工具脚本下载
root@k8s-master1:~# ./ezdown -D
##查看docker 版本
root@k8s-master1:~# docker version
## 下载额外容器镜像
root@k8s-master1:~# ./ezdown -X
## 下载离线文件
root@k8s-master1:~# ./ezdown -P
```
### 2.1.2 设置ssh免密登录部署服务器
```bash
## 主控机设置密钥
root@k8s-master1:~# ssh-keygen
## 设置免密登录
root@k8s-master1:~# ssh-copy-id 192.168.50.101
root@k8s-master1:~# ssh-copy-id 192.168.50.102
root@k8s-master1:~# ssh-copy-id 192.168.50.103
root@k8s-master1:~# ssh-copy-id 192.168.50.201
root@k8s-master1:~# ssh-copy-id 192.168.50.202
```
## 2.2 启动kubeasz容器
```bash
root@k8s-master1:~# ./ezdown -S
2023-01-24 16:36:36 INFO Action begin: start_kubeasz_docker
2023-01-24 16:36:36 INFO try to run kubeasz in a container
2023-01-24 16:36:36 DEBUG get host IP: 192.168.50.201
2023-01-24 16:36:36 DEBUG create a soft link '/usr/bin/python'
deb75009e9aec402a0637d067f67db7c6b716af9d25dd9212bef25879099073c
2023-01-24 16:36:36 INFO Action successed: start_kubeasz_docker

```
## 2.3 进入kubeasz创建集群k8s-01
```bash
root@k8s-master1:~# docker exec -it kubeasz bash
bash-5.1# ezctl new k8s-01

2023-01-24 15:30:44 DEBUG generate custom cluster files in /etc/kubeasz/clusters/k8s-01
2023-01-24 15:30:44 DEBUG set versions
2023-01-24 15:30:44 DEBUG cluster k8s-01: files successfully created.
2023-01-24 15:30:44 INFO next steps 1: to config '/etc/kubeasz/clusters/k8s-01/hosts'
2023-01-24 15:30:44 INFO next steps 2: to config '/etc/kubeasz/clusters/k8s-01/config.yml'
bash-5.1# vi /etc/kubeasz/clusters/k8s-01/hosts
[etcd]
192.168.50.101
192.168.50.102
192.168.50.103

[kube_master]
192.168.50.201
192.168.50.202

[kube_node]
192.168.50.101
192.168.50.102
192.168.50.103


[harbor]

[ex_lb]

[chrony]

[all:vars]
SECURE_PORT="6443"

CONTAINER_RUNTIME="containerd"

CLUSTER_NETWORK="calico"

PROXY_MODE="ipvs"

SERVICE_CIDR="10.68.0.0/16"

CLUSTER_CIDR="172.20.0.0/16"

NODE_PORT_RANGE="30000-32767"

CLUSTER_DNS_DOMAIN="cluster.local"

bin_dir="/opt/kube/bin"

base_dir="/etc/kubeasz"

cluster_dir="{{ base_dir }}/clusters/k8s-01"

ca_dir="/etc/kubernetes/ssl"
```
## 2.3 开始安装
```bash
root@k8s-master1:~# docker exec -it kubeasz bash
## 一键安装，等价于执行docker exec -it kubeasz ezctl setup k8s-01 all
bash-5.1# ezctl setup k8s-01 all

## 分步安装 需要分别执行01-07的yml
## 01-创建证书和环境准备
bash-5.1# ezctl setup k8s-01 01
## 02-安装etcd集群
bash-5.1# ezctl setup k8s-01 02
## 03-安装容器运行时（docker or containerd）
bash-5.1# ezctl setup k8s-01 03
## 04-安装kube_master节点
bash-5.1# ezctl setup k8s-01 04
## 05-安装kube_node节点
bash-5.1# ezctl setup k8s-01 05
## 06-安装网络组件
bash-5.1# ezctl setup k8s-01 06
## 07-安装集群主要插件
bash-5.1# ezctl setup k8s-01 07

## 安装成功输出信息
**********************************************************************************************************************************
192.168.50.101             : ok=73   changed=36   unreachable=0    failed=0    skipped=138  rescued=0    ignored=1
192.168.50.102             : ok=71   changed=35   unreachable=0    failed=0    skipped=138  rescued=0    ignored=1
192.168.50.103             : ok=71   changed=35   unreachable=0    failed=0    skipped=138  rescued=0    ignored=1
192.168.50.201             : ok=91   changed=50   unreachable=0    failed=0    skipped=164  rescued=0    ignored=1
192.168.50.202             : ok=83   changed=58   unreachable=0    failed=0    skipped=138  rescued=0    ignored=1
localhost                  : ok=32   changed=22   unreachable=0    failed=0    skipped=53   rescued=0    ignored=0
```
## 2.5 验证安装
```bash
## 查看各节点就绪 (Ready) 状态、角色、运行时间以及版本号
root@k8s-master1:~#  kubectl get nodes -o wide
NAME             STATUS                     ROLES    AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
192.168.50.101   Ready                      node     7m2s    v1.26.0   192.168.50.101   <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.8
192.168.50.102   Ready                      node     7m2s    v1.26.0   192.168.50.102   <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.8
192.168.50.103   Ready                      node     7m2s    v1.26.0   192.168.50.103   <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.8
192.168.50.201   Ready,SchedulingDisabled   master   8m13s   v1.26.0   192.168.50.201   <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.8
192.168.50.202   Ready,SchedulingDisabled   master   8m13s   v1.26.0   192.168.50.202   <none>        Ubuntu 22.04.1 LTS   5.15.0-53-generic   containerd://1.6.8
## 查看scheduler/controller-manager/etcd等组件
root@k8s-master1:~# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
etcd-0               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true","reason":""}
controller-manager   Healthy   ok
## 查看kubernetes master(apiserver)组件
root@k8s-master1:~# kubectl cluster-info
Kubernetes control plane is running at https://192.168.50.201:6443
CoreDNS is running at https://192.168.50.201:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
KubeDNSUpstream is running at https://192.168.50.201:6443/api/v1/namespaces/kube-system/services/kube-dns-upstream:dns/proxy
kubernetes-dashboard is running at https://192.168.50.201:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
## 有集群pod状态，默认已安装网络插件flannel、coredns、metrics-server等
root@k8s-master1:~# kubectl get po --all-namespaces
NAMESPACE     NAME                                         READY   STATUS    RESTARTS        AGE
kube-system   calico-kube-controllers-89b744d6c-lb25g      1/1     Running   0               9m6s
kube-system   calico-node-56xrl                            1/1     Running   1 (7m20s ago)   9m7s
kube-system   calico-node-g5g7g                            1/1     Running   2 (7m27s ago)   9m7s
kube-system   calico-node-gjbpm                            1/1     Running   4 (6m ago)      9m7s
kube-system   calico-node-lzrdv                            1/1     Running   2 (6m37s ago)   9m7s
kube-system   calico-node-p5n2p                            1/1     Running   0               9m7s
kube-system   coredns-6665999d97-j8xq9                     1/1     Running   0               4m23s
kube-system   dashboard-metrics-scraper-57566685b4-8c8mt   1/1     Running   0               4m14s
kube-system   kubernetes-dashboard-57db9bfd5b-tdhhl        1/1     Running   0               4m14s
kube-system   metrics-server-6bd9f986fc-z69tv              1/1     Running   0               4m18s
kube-system   node-local-dns-49hps                         1/1     Running   0               4m21s
kube-system   node-local-dns-csh6s                         1/1     Running   0               4m21s
kube-system   node-local-dns-dpnds                         1/1     Running   0               4m21s
kube-system   node-local-dns-lfw4v                         1/1     Running   0               4m21s
kube-system   node-local-dns-sfkgz                         1/1     Running   0               4m21s
```
## 2.6 访问dashboard
```bash
## 查看dashboard nodeport端口号
root@k8s-master1:~# kubectl get svc -A
NAMESPACE     NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes                  ClusterIP   10.68.0.1      <none>        443/TCP                  14m
kube-system   dashboard-metrics-scraper   ClusterIP   10.68.48.12    <none>        8000/TCP                 7m11s
kube-system   kube-dns                    ClusterIP   10.68.0.2      <none>        53/UDP,53/TCP,9153/TCP   7m20s
kube-system   kube-dns-upstream           ClusterIP   10.68.229.22   <none>        53/UDP,53/TCP            7m19s
kube-system   kubernetes-dashboard        NodePort    10.68.77.17    <none>        443:31986/TCP            7m11s
kube-system   metrics-server              ClusterIP   10.68.7.249    <none>        443/TCP                  7m15s
kube-system   node-local-dns              ClusterIP   None           <none>        9253/TCP                 7m18s
## 从master节点查看token并使用token登录（这里为了方便，我们可以直接使用admin-user的token）
root@k8s-master1:~# kubectl get po -A
NAMESPACE     NAME                                         READY   STATUS    RESTARTS      AGE
kube-system   calico-kube-controllers-89b744d6c-lb25g      1/1     Running   0             14m
kube-system   calico-node-56xrl                            1/1     Running   1 (12m ago)   14m
kube-system   calico-node-g5g7g                            1/1     Running   2 (13m ago)   14m
kube-system   calico-node-gjbpm                            1/1     Running   4 (11m ago)   14m
kube-system   calico-node-lzrdv                            1/1     Running   2 (12m ago)   14m
kube-system   calico-node-p5n2p                            1/1     Running   0             14m
kube-system   coredns-6665999d97-j8xq9                     1/1     Running   0             9m59s
kube-system   dashboard-metrics-scraper-57566685b4-8c8mt   1/1     Running   0             9m50s
kube-system   kubernetes-dashboard-57db9bfd5b-tdhhl        1/1     Running   0             9m50s
kube-system   metrics-server-6bd9f986fc-z69tv              1/1     Running   0             9m54s
kube-system   node-local-dns-49hps                         1/1     Running   0             9m57s
kube-system   node-local-dns-csh6s                         1/1     Running   0             9m57s
kube-system   node-local-dns-dpnds                         1/1     Running   0             9m57s
kube-system   node-local-dns-lfw4v                         1/1     Running   0             9m57s
kube-system   node-local-dns-sfkgz                         1/1     Running   0             9m57s
root@k8s-master1:~# kubectl get secret -n kube-system
NAME                              TYPE                                  DATA   AGE
admin-user                        kubernetes.io/service-account-token   3      10m
calico-etcd-secrets               Opaque                                3      14m
dashboard-read-user               kubernetes.io/service-account-token   3      10m
kubernetes-dashboard-certs        Opaque                                0      10m
kubernetes-dashboard-csrf         Opaque                                1      10m
kubernetes-dashboard-key-holder   Opaque                                2      10m
root@k8s-master1:~# kubectl describe secret -n kube-system admin-user
Name:         admin-user
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: f3642cb3-e631-4588-bc27-a212b31e991a

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1310 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6ImdTcnB5aHBkY1N4UUU0Y3B1ZlFPb2tNV2dycWp2aUdyb1B0a0w4Y00zTXcifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmMzY0MmNiMy1lNjMxLTQ1ODgtYmMyNy1hMjEyYjMxZTk5MWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.ls2vEyoxz7VQlh4LooTx5EgRVpQ3Fo2mxryZFF-uO1xhGQmX-hthNk9b-Mh7wQlWc_pEaJm4HonVUPqSKx_W71Ij4o4aYF3Et1i466HWWDulHRZ8c0Qnotfo__5EJKfOW1gMPtFDGM0jkJaqne68vCiDmCsvmUTSURp48lMdSZTZFVbQ_qtaw_vI37MyN58SnXWAApq9hErGtkFBg39YAFOP_unlIJ6VCNGL_GmcGDick8kyhNVDJlaYmje9iHWtuf-jExzBLnc2ZNjZlNk8zTLObjsot43cZ5C86KU_0-zB3xtl3yXNj6EAU_7SnAi4Gs1jB9II8dDZl-y0vgC6Hg
```
* 登录界面
![login](pictures/kubernetes-dashboard-login.png)
* POD信息
![pod](pictures/kubernetes-dashboards-info-01.png)

## 2.7 恢复 Master 调度
* 默认情况下，pod 节点不会分配到 master 节点，可以通过如下命令让 master 节点恢复调度，这样后续master也可以运行pod了。
```bash
root@k8s-master1:~# kubectl get node
NAME             STATUS                     ROLES    AGE   VERSION
192.168.50.101   Ready                      node     20m   v1.26.0
192.168.50.102   Ready                      node     20m   v1.26.0
192.168.50.103   Ready                      node     20m   v1.26.0
192.168.50.201   Ready,SchedulingDisabled   master   21m   v1.26.0
192.168.50.202   Ready,SchedulingDisabled   master   21m   v1.26.0
root@k8s-master1:~# kubectl uncordon 192.168.50.201 192.168.50.202
node/192.168.50.201 uncordoned
node/192.168.50.202 uncordoned
root@k8s-master1:~# kubectl get node
NAME             STATUS   ROLES    AGE   VERSION
192.168.50.101   Ready    node     20m   v1.26.0
192.168.50.102   Ready    node     20m   v1.26.0
192.168.50.103   Ready    node     20m   v1.26.0
192.168.50.201   Ready    master   21m   v1.26.0
192.168.50.202   Ready    master   21m   v1.26.0
```

# 3. 在集群上编排运行demoapp，并使用Service完成Pod发现和服务发布
```bash
## 创建一个deployment demoapp应用，指定副本为3
root@k8s-master1:~# k create deployment demoapp --image=ikubernetes/demoapp:v1.0 --replicas=3
deployment.apps/demoapp created
## 查看应用的pod
root@k8s-master1:~# k get pod
NAME                      READY   STATUS    RESTARTS   AGE
demoapp-75f59c894-kwvmh   1/1     Running   0          7s
demoapp-75f59c894-s9ldl   1/1     Running   0          7s
demoapp-75f59c894-wj2lv   1/1     Running   0          7s
## 创建一个demoapp service 绑定应用demoapp
root@k8s-master1:~# k create service nodeport demoapp --tcp=80:80
service/demoapp created
## 查看service
root@k8s-master1:~# k get svc

NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
demoapp      NodePort    10.111.26.59   <none>        80:31965/TCP   6s
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        13h
## 查看endpoints信息
root@k8s-master1:~# k get ep
NAME         ENDPOINTS                                   AGE
demoapp      10.244.1.3:80,10.244.2.4:80,10.244.3.3:80   14s
kubernetes   192.168.50.201:6443                         13h
kubernetes   192.168.50.201:6443                         13h
## 请求service IP 测试服务
root@k8s-master1:~# while true; do curl 10.111.26.59 ; sleep 1;done
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-s9ldl, ServerIP: 10.244.3.3!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-s9ldl, ServerIP: 10.244.3.3!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-s9ldl, ServerIP: 10.244.3.3!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-wj2lv, ServerIP: 10.244.2.4!
iKubernetes demoapp v1.0 !! ClientIP: 10.244.0.0, ServerName: demoapp-75f59c894-kwvmh, ServerIP: 10.244.1.3!
```
# 4. 要求以配置文件的方式，在集群上编排运行nginx，并使用Service完成Pod发现和服务发布
```bash
## 生成nginx应用 yaml文件
root@k8s-master1:~# kubectl create deployment my-nginx --image=nginx:latest --replicas=2 --dry-run=client -o yaml > deploy-my-nginx.yaml
## 编辑yaml文件
root@k8s-master1:~# vi deploy-my-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-nginx
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
## 应用yaml文件
root@k8s-master1:~# k apply -f deploy-my-nginx.yaml
## 查看pod
root@k8s-master1:~# k get pod
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-55d484674c-5bczw   1/1     Running   0          46s
my-nginx-55d484674c-zb8hg   1/1     Running   0          46s
## 创建my-nginx service yaml文件
root@k8s-master1:~# k create service nodeport my-nginx --tcp=80:80 --dry-run=client -o yaml >service-my-nginx.yaml
## 编辑yaml文件
root@k8s-master1:~# vi service-my-nginx.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-nginx
  name: my-nginx
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: my-nginx
  type: NodePort
## 应用my-nginx service yaml文件
root@k8s-master1:~# k apply -f service-my-nginx.yaml
service/my-nginx created
## 查看service 信息
root@k8s-master1:~# k get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        13h
my-nginx     NodePort    10.96.21.132   <none>        80:32064/TCP   5s
## 查看endpooint信息
root@k8s-master1:~# k get ep
NAME         ENDPOINTS                     AGE
kubernetes   192.168.50.201:6443           13h
my-nginx     10.244.1.4:80,10.244.3.4:80   9s
## 访问cluster IP 测试
root@k8s-master1:~# curl 10.96.21.132
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
# 5. 扩展作业：要求以配置文件的方式，在集群上编排运行wordpress和mysql，并使用Service完成Pod发现和服务发布
    提示：使用变量的方式的为wordpress指定要使用mysql服务器地址、数据库名称、用户名称和用户密码；
```bash
## 编辑deploy和service db yaml文件
root@k8s-master1:~# vi wordpress-db.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deploy
  labels:
    app: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        imagePullPolicy: IfNotPresent
        args:
        - --default_authentication_plugin=mysql_native_password
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
        ports:
        - containerPort: 3306
          name: dbport
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: Ygc20220124@
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          value: wordpress
        volumeMounts:
        - name: db
          mountPath: /var/lib/mysql
      volumes:
      - name: db
        hostPath:
          path: /var/lib/mysql

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
  - name: mysqlport
    protocol: TCP
    port: 3306
    targetPort: dbport
## 应用yaml文件
root@k8s-master1:~# k apply -f wordpress-db.yaml
deployment.apps/mysql-deploy created
service/mysql created
## 查看pod状态
root@k8s-master1:~# k get pod
NAME                            READY   STATUS    RESTARTS   AGE
mysql-deploy-5bcb5d8fcd-lk9wp   1/1     Running   0          42s
## 查看service
root@k8s-master1:~# k get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP    14h
mysql        ClusterIP   10.103.1.149   <none>        3306/TCP   57s
##查看endpoint
root@k8s-master1:~# k get ep
NAME         ENDPOINTS             AGE
kubernetes   192.168.50.201:6443   14h
mysql        10.244.1.5:3306       72s
## 查看mysql service详细信息
root@k8s-master1:~# k describe svc mysql
Name:              mysql
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=mysql
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.103.1.149
IPs:               10.103.1.149
Port:              mysqlport  3306/TCP
TargetPort:        dbport/TCP
Endpoints:         10.244.1.5:3306
Session Affinity:  None
Events:            <none>
## 编辑wordpress yaml文件
root@k8s-master1:~# vi wordpress.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deploy
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: wdport
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql.default.svc:3306
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          value: wordpress
## 应用yaml文件
root@k8s-master1:~# k apply -f wordpress.yaml
## 查看pod
root@k8s-master1:~# k get pod
NAME                                READY   STATUS    RESTARTS   AGE
mysql-deploy-5bcb5d8fcd-lk9wp       1/1     Running   0          6m35s
wordpress-deploy-67b75bddb9-z5pc6   1/1     Running   0          69s
## 编辑增加service 部分配置
root@k8s-master1:~# vi wordpress.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deploy
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: wdport
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql.default.svc:3306
        - name: WORDPRESS_DB_USER
          value: wordpress
        - name: WORDPRESS_DB_PASSWORD
          value: wordpress

---
apiVersion: v1
kind: Service
metadata:
  name: wordpress
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
  - name: wordpressport
    protocol: TCP
    port: 80
    targetPort: wdport
## 应用yaml文件
root@k8s-master1:~# kubectl apply -f wordpress.yaml
deployment.apps/wordpress-deploy unchanged
service/wordpress created
## 查看service
root@k8s-master1:~# k get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        14h
mysql        ClusterIP   10.103.1.149    <none>        3306/TCP       9m6s
wordpress    NodePort    10.111.117.60   <none>        80:30734/TCP   29s
```
* wordpress 安装界面
![install](pictures/wordpress-install.png)
* 填写blog相关信息
![blog](pictures/wordpress-install-02.png)
* 安装完成
![success](pictures/wordpress-install-success.png)