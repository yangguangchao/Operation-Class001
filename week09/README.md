# 1. 梳理ceph的各组件及功能
* 一个 ceph 集群的组成部分：
    * 若干的 Ceph OSD(对象存储守护程序)
    * 至少需要一个 Ceph Monitors 监视器(1,3,5,7...)
    * 两个或以上的 Ceph 管理器 managers，运行 Ceph 文件系统客户端时 还需要高可用的 Ceph Metadata Server(文件系统元数据服务器)。
* RADOS cluster:由多台 host 存储服务器组成的 ceph 集群
* OSD(Object Storage Daemon):每台存储服务器的磁盘组成的存储空间
* Mon(Monitor):ceph 的监视器,维护 OSD 和 PG 的集群状态，一个 ceph 集群至少要有一个 mon，可以是一三五七等等这样的奇数个。
* Mgr(Manager):负责跟踪运行时指标和 Ceph 集群的当前状态，包括存储利用率，当前性 能指标和系统负载等。
## 1.1 Monitor(ceph-mon) ceph 监视器
* 在一个主机上运行的一个守护进程，用于维护集群状态映射(maintains maps of the cluster state)，比如 ceph 集群中有多少存储池、每个存储池有多少 PG 以及存储池和 PG
的 映 射 关 系 等 ， monitor map, manager map, the OSD map, the MDS map, and the CRUSH map，这些映射是 Ceph 守护程序相互协调所需的关键群集状态，此外监视器还负 责管理守护程序和客户端之间的身份验证(认证使用 cephX 协议)。通常至少需要三个监视器 才能实现冗余和高可用性。
## 1.2 Managers(ceph-mgr)的功能
* 在一个主机上运行的一个守护进程，Ceph Manager 守护程序(ceph-mgr)负责跟踪 运行时指标和 Ceph 集群的当前状态，包括存储利用率，当前性能指标和系统负载。Ceph Manager 守护程序还托管基于 python 的模块来管理和公开 Ceph 集群信息，包括基于 Web 的 Ceph 仪表板和 REST API。高可用性通常至少需要两个管理器。
## 1.3 Ceph OSDs(对象存储守护程序 ceph-osd)
* 提供存储数据，操作系统上的一个磁盘就是一个 OSD 守护程序，OSD 用于处理 ceph 集群数据复制，恢复，重新平衡，并通过检查其他 Ceph OSD 守护程序的心跳来向 Ceph 监视器和
## 1.4 MDS(ceph 元数据服务器 ceph-mds):
* 代表 ceph 文件系统(NFS/CIFS)存储元数据，(即 Ceph 块设备和 Ceph 对象存储不使用 MDS)
## 1.5 Ceph 的管理节点
* ceph 的常用管理接口是一组命令行工具程序，例如 rados、ceph、rbd 等命令，ceph 管 理员可以从某个特定的 ceph-mon 节点执行管理操作
* 推荐使用部署专用的管理节点对 ceph 进行配置管理、升级与后期维护，方便后期权限管 理，管理节点的权限只对管理人员开放，可以避免一些不必要的误操作的发生。  

# 2. 基于ceph-deploy部署ceph集群
* 服务器列表
```bash
11.0.1.100 ceph-deploy.example.local  ceph-deploy
11.0.1.101 ceph-node1.example.local   ceph-node1   
11.0.1.102 ceph-node2.example.local   ceph-node2  
11.0.1.103 ceph-node3.example.local   ceph-node3 
11.0.1.104 ceph-node4.example.local   ceph-node4
11.0.1.105 ceph-mon1.example.local    ceph-mon1   
11.0.1.106 ceph-mon2.example.local    ceph-mon2  
11.0.1.107 ceph-mon3.example.local    ceph-mon3  
11.0.1.108 ceph-mgr1.example.local    ceph-mgr1   
11.0.1.109 ceph-mgr2.example.local    ceph-mgr2    
```
## 2.1 部署环境
### 2.1.1 OSD 存储服务器
四台服务器作为 ceph 集群 OSD 存储服务器，每台服务器支持两个网络，public 网络针 对客户端访问，cluster 网络用于集群管理及数据同步，每台三块或以上的磁盘
```bash
11.0.1.101/192.168.50.101    ceph-node1   
11.0.1.102/192.168.50.102    ceph-node2  
11.0.1.103/192.168.50.103    ceph-node3 
11.0.1.104/192.168.50.104    ceph-node4

/dev/sdb /dev/sdc /dev/sdd /dev/sde #4T
/dev/nvme0n1 #2T
```
### 2.1.2 Mon 监视服务器
三台服务器作为 ceph 集群 Mon 监视服务器，每台服务器可以和 ceph 集群的 cluster 网 络通信。
```bash
11.0.1.105/192.168.50.105    ceph-mon1   
11.0.1.106/192.168.50.106    ceph-mon2  
11.0.1.107/192.168.50.107    ceph-mon3  
```
### 2.1.3 mgr 管理服务器
两个 ceph-mgr 管理服务器，可以和 ceph 集群的 cluster 网络通信。
```bash
11.0.1.108/192.168.50.108    ceph-mgr1   
11.0.1.109/192.168.50.109   ceph-mgr2  
```
### 2.1.4 Ceph-deploy服务器
一个服务器用于部署 ceph 集群即安装 Ceph-deploy，也可以和 ceph-mgr 等复用。
```bash
11.0.1.100/192.168.50.100  ceph-deploy
```

## 2.2 仓库准备
* 各节点配置 ceph yum 仓库
* 导入 key 文件
```bash
## 支持 https 镜像仓库源
root@ceph-deploy:~# apt install -y apt-transport-https ca-certificates curl software-properties-common
## 导入key
root@ceph-deploy:~# wget -q -O- 'https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc' | sudo apt-key add -
## 各节点配置 epel 仓库
root@ceph-deploy:~# echo "deb https://mirrors.tuna.tsinghua.edu.cn/ceph/debian-pacific bionic main" >> /etc/apt/sources.list
root@ceph-deploy:~# cat /etc/apt/sources.list
#

# deb cdrom:[Ubuntu-Server 18.04.6 LTS _Bionic Beaver_ - Release amd64 (20210915.2)]/ bionic main restricted

#deb cdrom:[Ubuntu-Server 18.04.6 LTS _Bionic Beaver_ - Release amd64 (20210915.2)]/ bionic main restricted

# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted

## Major bug fix updates produced after the final release of the
## distribution.
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic universe
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic universe
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates universe
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates universe

## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team, and may not be under a free licence. Please satisfy yourself as to
## your rights to use the software. Also, please note that software in
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates multiverse

## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse

## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu bionic partner
# deb-src http://archive.canonical.com/ubuntu bionic partner

deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security main restricted
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security main restricted
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security universe
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security universe
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ceph/debian-pacific bionic main
root@ceph-deploy:~# apt update
```
## 2.3 创建 ceph 集群部署用户 cephadmin
* 推荐使用指定的普通用户部署和运行 ceph 集群，普通用户只要能以非交互方式执行 sudo 命令执行一些特权命令即可，新版的 ceph-deploy 可以指定包含 root 的在内只要可以执 行 sudo 命令的用户，不过仍然推荐使用普通用户，ceph 集群安装完成后会自动创建 ceph 用户(ceph 集群默认会使用 ceph 用户运行各服务进程,如 ceph-osd 等)，因此推荐 使用除了 ceph 用户之外的比如 cephuser、cephadmin 这样的普通用户去部署和 管理 ceph 集群。
* cephadmin 仅用于通过 ceph-deploy 部署和管理 ceph 集群的时候使用，比如首次初始 化集群和部署集群、添加节点、删除节点等，ceph 集群在 node 节点、mgr 等节点会使用 ceph 用户启动服务进程。

```bash
##在包含 ceph-deploy 节点的存储节点、mon 节点和 mgr 节点等创建 cephadmin 用户。
root@ceph-deploy:~# groupadd -r -g 2088 cephadmin && useradd -r -m -s /bin/bash -u 2088 -g 2088 cephadmin && echo cephadmin:123456 | chpasswd
## 各服务器允许 cephadmin 用户以 sudo 执行特权命令:
root@ceph-deploy:~# echo "cephadmin ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
## 配置免秘钥登录
## 在 ceph-deploy 节点配置秘钥分发，允许 cephadmin 用户以非交互的方式登录到各 ceph node/mon/mgr 节点进行集群部署及管理操作，即在 ceph-deploy 节点生成秘钥对，然后分 发公钥到各被管理节点
root@ceph-node1:~# su - cephadmin
cephadmin@ceph-node1:~$ ssh-keygen
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.100
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.101
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.102
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.103
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.104
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.105
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.106
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.107
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.108
cephadmin@ceph-deploy:~$ ssh-copy-id cephadmin@11.0.1.109
```
## 2.4 配置主机名解析
* 各节点配置主机名解析
```bash
cephadmin@ceph-deploy:~$  sudo vim /etc/hosts
11.0.1.100      ceph-deploy.example.local       ceph-deploy
11.0.1.101      ceph-node1.example.local        ceph-node1
11.0.1.102      ceph-node2.example.local        ceph-node2
11.0.1.103      ceph-node3.example.local        ceph-node3
11.0.1.104      ceph-node4.example.local        ceph-node4
11.0.1.105      ceph-mon1.example.local         ceph-mon1
11.0.1.106      ceph-mon2.example.local         ceph-mon2
11.0.1.107      ceph-mon3.example.local         ceph-mon3
11.0.1.108      ceph-mgr1.example.local         ceph-mgr1
11.0.1.109      ceph-mgr2.example.local         ceph-mgr2
```
## 2.5 安装 ceph 部署工具
```bash
##安装 python2
root@ceph-deploy:~$ sudo  apt install python-pip
##使用 python2 的 pip 安装 ceph-deploy
root@ceph-deploy:~# pip2 install ceph-deploy
```
## 2.6 初始化 mon 节点
* 在管理节点初始化 mon 节点
```bash
cephadmin@ceph-deploy:~$  mkdir ceph-cluster
cephadmin@ceph-deploy:~$ cd ceph-cluster/
```
* 初始化 mon 节点过程如下
```bash
## Ubuntu 各服务器需要单独安装 Python2:
root@ceph-mon1:~# apt install python2.7 -y
root@ceph-mon1:~# ln -sv /usr/bin/python2.7 /usr/bin/python2
'/usr/bin/python2' -> '/usr/bin/python2.7'
cephadmin@ceph-deploy:~/ceph-cluster$  ceph-deploy new --cluster-network 11.0.1.0/24 --public-network 192.168.50.0/24 ceph-mon1.example.local ceph-mon2.example.local ceph-mon3.example.local
##验证初始化:
cephadmin@ceph-deploy:~/ceph-cluster$ ll
total 24
drwxrwxr-x 2 cephadmin cephadmin 4096 Dec 19 21:44 ./
drwxr-xr-x 6 cephadmin cephadmin 4096 Dec 19 21:36 ../
-rw-rw-r-- 1 cephadmin cephadmin  264 Dec 19 21:44 ceph.conf
-rw-rw-r-- 1 cephadmin cephadmin 4175 Dec 19 21:44 ceph-deploy-ceph.log
-rw------- 1 cephadmin cephadmin   73 Dec 19 21:44 ceph.mon.keyring
cephadmin@ceph-deploy:~/ceph-cluster$ cat ceph.conf
[global]
fsid = 97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
public_network = 192.168.50.0/24
cluster_network = 11.0.1.0/24
mon_initial_members = ceph-mon1, ceph-mon2, ceph-mon3
mon_host = 192.168.50.105,192.168.50.106,192.168.50.107
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```
## 2.7 初始化 ceph 存储节点
```bash
##初始化 node 节点
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy install --no-adjust-repos --nogpgcheck ceph-node1 ceph-node2 ceph-node3 ceph-node4
......
[ceph-node4][DEBUG ] Processing triggers for systemd (237-3ubuntu10.52) ...
[ceph-node4][DEBUG ] Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
[ceph-node4][DEBUG ] Processing triggers for ureadahead (0.100.0-21) ...
[ceph-node4][DEBUG ] Processing triggers for libc-bin (2.27-3ubuntu1.4) ...
[ceph-node4][INFO  ] Running command: sudo ceph --version
[ceph-node4][DEBUG ] ceph version 16.2.10 (45fa1a083152e41a408d15505f594ec5f1b4fe17) pacific (stable)
```
## 2.8 安装 ceph-mon 服务
* 在各 mon 节点按照组件 ceph-mon,并通初始化 mon 节点，mon 节点 HA 还可以后期横向扩容。
### 2.8.1 安装 ceph-mon
```bash
root@ceph-mon1:~# apt-cache madison ceph-mon
  ceph-mon | 16.2.10-1bionic | https://mirrors.tuna.tsinghua.edu.cn/ceph/debian-pacific bionic/main amd64 Packages
  ceph-mon | 12.2.13-0ubuntu0.18.04.10 | https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-updates/main amd64 Packages
  ceph-mon | 12.2.13-0ubuntu0.18.04.10 | https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic-security/main amd64 Packages
  ceph-mon | 12.2.4-0ubuntu1 | https://mirrors.tuna.tsinghua.edu.cn/ubuntu bionic/main amd64 Packages
 root@ceph-mon1:~# apt install ceph-mon
 root@ceph-mon2:~# apt install ceph-mon
 root@ceph-mon3:~# apt install ceph-mon
```
### 2.8.2 ceph 集群添加 ceph-mon 服务
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ pwd
/home/cephadmin/ceph-cluster
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy mon create-initial
......
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.client.admin.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mds.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-mgr.keyring
[ceph_deploy.gatherkeys][INFO  ] keyring 'ceph.mon.keyring' already exists
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-osd.keyring
[ceph_deploy.gatherkeys][INFO  ] Storing ceph.bootstrap-rgw.keyring
[ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/tmpnSwSXz
```
### 2.8.3 验证 mon 节点
```bash
root@ceph-mon1:~# ps -ef | grep ceph-mon
ceph       13242       1  0 22:17 ?        00:00:00 /usr/bin/ceph-mon -f --cluster ceph --id ceph-mon1 --setuser ceph --setgroup ceph
root       13799    1297  0 22:20 pts/0    00:00:00 grep --color=auto ceph-mon
You have new mail in /var/mail/root
```
## 2.9 分发 admin 秘钥
* 在 ceph-deploy 节点把配置文件和 admin 密钥拷贝至 Ceph 集群需要执行 ceph 管理命令的 节点，从而不需要后期通过 ceph 命令对 ceph 集群进行管理配置的时候每次都需要指定 ceph-mon 节点地址和 ceph.client.admin.keyring 文件,另外各 ceph-mon 节点也需要同步 ceph 的集群配置文件与认证文件
```bash
## 如果在 ceph-deploy 节点管理集群:
cephadmin@ceph-deploy:~/ceph-cluster$ sudo apt install ceph-common
root@ceph-node1:~# apt install ceph-common -y
root@ceph-node2:~# apt install ceph-common -y
root@ceph-node3:~# apt install ceph-common -y
root@ceph-node4:~# apt install ceph-common -y
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy admin ceph-node1 ceph-node2 ceph-node3 ceph-node4
## 到 node 节点验证 key 文件
root@ceph-node1:~# ll /etc/ceph/
total 20
drwxr-xr-x  2 root root 4096 Dec 19 22:24 ./
drwxr-xr-x 95 root root 4096 Dec 19 22:04 ../
-rw-------  1 root root  151 Dec 19 22:24 ceph.client.admin.keyring
-rw-r--r--  1 root root  316 Dec 19 22:24 ceph.conf
-rw-r--r--  1 root root   92 Jul 22 01:38 rbdmap
-rw-------  1 root root    0 Dec 19 22:24 tmpy2rJ7b

## 认证文件的属主和属组为了安全考虑，默认设置为了 root 用户和 root 组，如果需要 ceph 用户也能执行 ceph 命令，那么就需要对 ceph 用户进行授权，
root@ceph-node1:~# setfacl -m u:cephadmin:rw /etc/ceph/ceph.client.admin.keyring
root@ceph-node2:~# setfacl -m u:cephadmin:rw /etc/ceph/ceph.client.admin.keyring
root@ceph-node3:~# setfacl -m u:cephadmin:rw /etc/ceph/ceph.client.admin.keyring
root@ceph-node4:~# setfacl -m u:cephadmin:rw /etc/ceph/ceph.client.admin.keyring
```
## 2.10 部署 ceph-mgr 节点
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy mgr --help
usage: ceph-deploy mgr [-h] {create} ...

Ceph MGR daemon management

positional arguments:
  {create}
    create    Deploy Ceph MGR on remote host(s)

optional arguments:
  -h, --help  show this help message and exit
## 初始化 ceph-mgr 节点:
root@ceph-mgr1:~# apt install ceph-mgr -y
root@ceph-mgr2:~# apt install ceph-mgr -y
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy mgr create ceph-mgr1 ceph-mgr2
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephadmin/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/local/bin/ceph-deploy mgr create ceph-mgr1 ceph-mgr2
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  mgr                           : [('ceph-mgr1', 'ceph-mgr1'), ('ceph-mgr2', 'ceph-mgr2')]
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f9022091f50>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mgr at 0x7f90224e7550>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mgr][DEBUG ] Deploying mgr, cluster ceph hosts ceph-mgr1:ceph-mgr1 ceph-mgr2:ceph-mgr2
The authenticity of host 'ceph-mgr1 (11.0.1.108)' can't be established.
ECDSA key fingerprint is SHA256:qW05ZuB6zoP0WiOSVa/KNK8W3LKJwLEf93Lj847HpUo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ceph-mgr1' (ECDSA) to the list of known hosts.
[ceph-mgr1][DEBUG ] connection detected need for sudo
[ceph-mgr1][DEBUG ] connected to host: ceph-mgr1
[ceph-mgr1][DEBUG ] detect platform information from remote host
[ceph-mgr1][DEBUG ] detect machine type
[ceph_deploy.mgr][INFO  ] Distro info: Ubuntu 18.04 bionic
[ceph_deploy.mgr][DEBUG ] remote host will use systemd
[ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to ceph-mgr1
[ceph-mgr1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-mgr1][WARNIN] mgr keyring does not exist yet, creating one
[ceph-mgr1][DEBUG ] create a keyring file
[ceph-mgr1][DEBUG ] create path recursively if it doesn't exist
[ceph-mgr1][INFO  ] Running command: sudo ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.ceph-mgr1 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-ceph-mgr1/keyring
[ceph-mgr1][INFO  ] Running command: sudo systemctl enable ceph-mgr@ceph-mgr1
[ceph-mgr1][WARNIN] Created symlink /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-mgr1.service → /lib/systemd/system/ceph-mgr@.service.
[ceph-mgr1][INFO  ] Running command: sudo systemctl start ceph-mgr@ceph-mgr1
[ceph-mgr1][INFO  ] Running command: sudo systemctl enable ceph.target
The authenticity of host 'ceph-mgr2 (11.0.1.109)' can't be established.
ECDSA key fingerprint is SHA256:qW05ZuB6zoP0WiOSVa/KNK8W3LKJwLEf93Lj847HpUo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ceph-mgr2' (ECDSA) to the list of known hosts.
[ceph-mgr2][DEBUG ] connection detected need for sudo
[ceph-mgr2][DEBUG ] connected to host: ceph-mgr2
[ceph-mgr2][DEBUG ] detect platform information from remote host
[ceph-mgr2][DEBUG ] detect machine type
[ceph_deploy.mgr][INFO  ] Distro info: Ubuntu 18.04 bionic
[ceph_deploy.mgr][DEBUG ] remote host will use systemd
[ceph_deploy.mgr][DEBUG ] deploying mgr bootstrap to ceph-mgr2
[ceph-mgr2][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-mgr2][WARNIN] mgr keyring does not exist yet, creating one
[ceph-mgr2][DEBUG ] create a keyring file
[ceph-mgr2][DEBUG ] create path recursively if it doesn't exist
[ceph-mgr2][INFO  ] Running command: sudo ceph --cluster ceph --name client.bootstrap-mgr --keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring auth get-or-create mgr.ceph-mgr2 mon allow profile mgr osd allow * mds allow * -o /var/lib/ceph/mgr/ceph-ceph-mgr2/keyring
[ceph-mgr2][INFO  ] Running command: sudo systemctl enable ceph-mgr@ceph-mgr2
[ceph-mgr2][WARNIN] Created symlink /etc/systemd/system/ceph-mgr.target.wants/ceph-mgr@ceph-mgr2.service → /lib/systemd/system/ceph-mgr@.service.
[ceph-mgr2][INFO  ] Running command: sudo systemctl start ceph-mgr@ceph-mgr2
[ceph-mgr2][INFO  ] Running command: sudo systemctl enable ceph.target

## 验证 ceph-mgr 节点
root@ceph-mgr1:~# ps -ef | grep ceph
root       13628       1  0 22:32 ?        00:00:00 /usr/bin/python3.6 /usr/bin/ceph-crash
ceph       19882       1  4 22:33 ?        00:00:05 /usr/bin/ceph-mgr -f --cluster ceph --id ceph-mgr1 --setuser ceph --setgroup ceph
root       20076    1271  0 22:35 pts/0    00:00:00 grep --color=auto ceph
```
## 2.11 ceph-deploy 管理 ceph 集群
* 在 ceph-deploy 节点配置一下系统环境，以方便后期可以执行 ceph 管理命令。
```bash
root@ceph-deploy:~# apt install ceph-common -y
## 推送证书给自己
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy admin ceph-deploy
```
## 2.12 测试 ceph 命令
```bash
## 授权
cephadmin@ceph-deploy:~/ceph-cluster$ sudo setfacl -m u:cephadmin:rw /etc/ceph/ceph.client.admin.keyring
cephadmin@ceph-deploy:~/ceph-cluster$ ceph -s
  cluster:
    id:     97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            clock skew detected on mon.ceph-mon2
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 3 daemons, quorum ceph-mon1,ceph-mon2,ceph-mon3 (age 23m)
    mgr: ceph-mgr1(active, since 8m), standbys: ceph-mgr2
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
## 禁用非安全模式通信
cephadmin@ceph-deploy:~/ceph-cluster$ ceph config set mon auth_allow_insecure_global_id_reclaim false

cephadmin@ceph-deploy:~/ceph-cluster$ ceph -s
  cluster:
    id:     97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
    health: HEALTH_WARN
            clock skew detected on mon.ceph-mon2
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 3 daemons, quorum ceph-mon1,ceph-mon2,ceph-mon3 (age 27m)
    mgr: ceph-mgr1(active, since 12m), standbys: ceph-mgr2
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```
## 2.13 初始化存储节点
* OSD 节点安装运行环境:
```bash
## 擦除磁盘之前通过 deploy 节点对 node 节点执行安装 ceph 基本运行环境
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy install --release pacific ceph-node1
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy install --release pacific ceph-node2
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy install --release pacific ceph-node3
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy install --release pacific ceph-node4
## 列出 ceph node 节点磁盘:
cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy disk list ceph-node1
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephadmin/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/local/bin/ceph-deploy disk list ceph-node1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  debug                         : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : list
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f8d72de36e0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  host                          : ['ceph-node1']
[ceph_deploy.cli][INFO  ]  func                          : <function disk at 0x7f8d72dae6d0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph-node1][DEBUG ] connection detected need for sudo
[ceph-node1][DEBUG ] connected to host: ceph-node1
[ceph-node1][DEBUG ] detect platform information from remote host
[ceph-node1][DEBUG ] detect machine type
[ceph-node1][DEBUG ] find the location of an executable
[ceph-node1][INFO  ] Running command: sudo fdisk -l
[ceph-node1][INFO  ] Disk /dev/nvme0n1: 2 TiB, 2250562863104 bytes, 4395630592 sectors
[ceph-node1][INFO  ] Disk /dev/sdd: 4 TiB, 4398046511104 bytes, 8589934592 sectors
[ceph-node1][INFO  ] Disk /dev/sdb: 4 TiB, 4398046511104 bytes, 8589934592 sectors
[ceph-node1][INFO  ] Disk /dev/sda: 60 GiB, 64424509440 bytes, 125829120 sectors
[ceph-node1][INFO  ] Disk /dev/sdc: 4 TiB, 4398046511104 bytes, 8589934592 sectors
[ceph-node1][INFO  ] Disk /dev/sde: 4 TiB, 4398046511104 bytes, 8589934592 sectors
## 使用 ceph-deploy disk zap 擦除各 ceph node 的 ceph 数据磁盘
cephadmin@ceph-deploy:~/ceph-cluster$ceph-deploy disk zap ceph-node1 /dev/sdb
ceph-deploy disk zap ceph-node1 /dev/sdc
ceph-deploy disk zap ceph-node1 /dev/sdd
ceph-deploy disk zap ceph-node1 /dev/sde
ceph-deploy disk zap ceph-node2 /dev/sdb
ceph-deploy disk zap ceph-node2 /dev/sdc
ceph-deploy disk zap ceph-node2 /dev/sdd
ceph-deploy disk zap ceph-node2 /dev/sde
ceph-deploy disk zap ceph-node3 /dev/sdb
ceph-deploy disk zap ceph-node3 /dev/sdc
ceph-deploy disk zap ceph-node3 /dev/sdd
ceph-deploy disk zap ceph-node3 /dev/sde
ceph-deploy disk zap ceph-node4 /dev/sdb
ceph-deploy disk zap ceph-node4 /dev/sdc
ceph-deploy disk zap ceph-node4 /dev/sdd
ceph-deploy disk zap ceph-node4 /dev/sde
ceph-deploy disk zap ceph-node1 /dev/nvme0n1
ceph-deploy disk zap ceph-node2 /dev/nvme0n1
ceph-deploy disk zap ceph-node3 /dev/nvme0n1
ceph-deploy disk zap ceph-node4 /dev/nvme0n1
```
## 2.14 添加 OSD
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ceph-deploy osd create ceph-node1 --data /dev/sdb
ceph-deploy osd create ceph-node1 --data /dev/sdc
ceph-deploy osd create ceph-node1 --data /dev/sdd
ceph-deploy osd create ceph-node1 --data /dev/sde
ceph-deploy osd create ceph-node2 --data /dev/sdb
ceph-deploy osd create ceph-node2 --data /dev/sdc
ceph-deploy osd create ceph-node2 --data /dev/sdd
ceph-deploy osd create ceph-node2 --data /dev/sde
ceph-deploy osd create ceph-node3 --data /dev/sdb
ceph-deploy osd create ceph-node3 --data /dev/sdc
ceph-deploy osd create ceph-node3 --data /dev/sdd
ceph-deploy osd create ceph-node3 --data /dev/sde
ceph-deploy osd create ceph-node4 --data /dev/sdb
ceph-deploy osd create ceph-node4 --data /dev/sdc
ceph-deploy osd create ceph-node4 --data /dev/sdd
ceph-deploy osd create ceph-node4 --data /dev/sde
## 查看集群状态
cephadmin@ceph-deploy:~/ceph-cluster$ ceph -s
  cluster:
    id:     97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-mon1,ceph-mon2,ceph-mon3 (age 52m)
    mgr: ceph-mgr1(active, since 37m), standbys: ceph-mgr2
    osd: 16 osds: 16 up (since 7m), 16 in (since 7m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   110 MiB used, 64 TiB / 64 TiB avail
    pgs:     1 active+clean
```
## 2.15 设置 OSD 服务自启动
* 默认就已经为自启动, node 节点添加完成后,可以重启服务器测试 node 服务器重启后，OSD 是否会自动启动。
```bash
root@ceph-node1:~#  ps -ef | grep osd
ceph       16608       1  0 22:58 ?        00:00:04 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph
ceph       18380       1  0 22:59 ?        00:00:04 /usr/bin/ceph-osd -f --cluster ceph --id 1 --setuser ceph --setgroup ceph
ceph       20150       1  0 22:59 ?        00:00:04 /usr/bin/ceph-osd -f --cluster ceph --id 2 --setuser ceph --setgroup ceph
ceph       21911       1  0 22:59 ?        00:00:05 /usr/bin/ceph-osd -f --cluster ceph --id 3 --setuser ceph --setgroup ceph
root       22461    1142  0 23:15 pts/0    00:00:00 grep --color=auto osd
root@ceph-node1:~# systemctl enable ceph-osd@0 ceph-osd@1 ceph-osd@2 ceph-osd@3
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@0.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@1.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@2.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@3.service → /lib/systemd/system/ceph-osd@.service.
root@ceph-node1:~# systemctl status ceph-osd@0
● ceph-osd@0.service - Ceph object storage daemon osd.0
   Loaded: loaded (/lib/systemd/system/ceph-osd@.service; indirect; vendor preset: enabled)
   Active: active (running) since Mon 2022-12-19 22:58:49 CST; 16min ago
 Main PID: 16608 (ceph-osd)
    Tasks: 59
   CGroup: /system.slice/system-ceph\x2dosd.slice/ceph-osd@0.service
           └─16608 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph

Dec 19 22:59:04 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:04 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
root@ceph-node1:~# systemctl status ceph-osd@1
● ceph-osd@1.service - Ceph object storage daemon osd.1
   Loaded: loaded (/lib/systemd/system/ceph-osd@.service; indirect; vendor preset: enabled)
   Active: active (running) since Mon 2022-12-19 22:59:04 CST; 16min ago
 Main PID: 18380 (ceph-osd)
    Tasks: 59
   CGroup: /system.slice/system-ceph\x2dosd.slice/ceph-osd@1.service
           └─18380 /usr/bin/ceph-osd -f --cluster ceph --id 1 --setuser ceph --setgroup ceph

Dec 19 22:59:08 ceph-node1 ceph-osd[18380]: 2022-12-19T22:59:08.296+0800 7f8fbc410700 -1 osd.1 0 waiting for initial osdmap
Dec 19 22:59:08 ceph-node1 ceph-osd[18380]: 2022-12-19T22:59:08.300+0800 7f8fb8ba2700 -1 osd.1 9 set_numa_affinity unable to identify public
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:18 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
root@ceph-node1:~# systemctl status ceph-osd@2
● ceph-osd@2.service - Ceph object storage daemon osd.2
   Loaded: loaded (/lib/systemd/system/ceph-osd@.service; indirect; vendor preset: enabled)
   Active: active (running) since Mon 2022-12-19 22:59:19 CST; 16min ago
 Main PID: 20150 (ceph-osd)
    Tasks: 59
   CGroup: /system.slice/system-ceph\x2dosd.slice/ceph-osd@2.service
           └─20150 /usr/bin/ceph-osd -f --cluster ceph --id 2 --setuser ceph --setgroup ceph

Dec 19 22:59:18 ceph-node1 systemd[1]: Starting Ceph object storage daemon osd.2...
Dec 19 22:59:19 ceph-node1 systemd[1]: Started Ceph object storage daemon osd.2.
Dec 19 22:59:21 ceph-node1 ceph-osd[20150]: 2022-12-19T22:59:21.436+0800 7f81240a0080 -1 osd.2 0 log_to_monitors {default=true}
Dec 19 22:59:23 ceph-node1 ceph-osd[20150]: 2022-12-19T22:59:23.332+0800 7f811ad1e700 -1 osd.2 0 waiting for initial osdmap
Dec 19 22:59:23 ceph-node1 ceph-osd[20150]: 2022-12-19T22:59:23.336+0800 7f81174b0700 -1 osd.2 14 set_numa_affinity unable to identify public
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:23: Unknown lvalue 'ProtectHostname' in section 'Service'
Dec 19 22:59:37 ceph-node1 systemd[1]: /lib/systemd/system/ceph-osd@.service:24: Unknown lvalue 'ProtectKernelLogs' in section 'Service'

root@ceph-node1:~# systemctl status ceph-osd@3
● ceph-osd@3.service - Ceph object storage daemon osd.3
   Loaded: loaded (/lib/systemd/system/ceph-osd@.service; indirect; vendor preset: enabled)
   Active: active (running) since Mon 2022-12-19 22:59:37 CST; 16min ago
  Process: 21907 ExecStartPre=/usr/libexec/ceph/ceph-osd-prestart.sh --cluster ${CLUSTER} --id 3 (code=exited, status=0/SUCCESS)
 Main PID: 21911 (ceph-osd)
    Tasks: 59
   CGroup: /system.slice/system-ceph\x2dosd.slice/ceph-osd@3.service
           └─21911 /usr/bin/ceph-osd -f --cluster ceph --id 3 --setuser ceph --setgroup ceph

Dec 19 22:59:37 ceph-node1 systemd[1]: Starting Ceph object storage daemon osd.3...
Dec 19 22:59:37 ceph-node1 systemd[1]: Started Ceph object storage daemon osd.3.
Dec 19 22:59:39 ceph-node1 ceph-osd[21911]: 2022-12-19T22:59:39.540+0800 7fc3763d4080 -1 osd.3 0 log_to_monitors {default=true}
Dec 19 22:59:40 ceph-node1 ceph-osd[21911]: 2022-12-19T22:59:40.800+0800 7fc36d052700 -1 osd.3 0 waiting for initial osdmap
Dec 19 22:59:40 ceph-node1 ceph-osd[21911]: 2022-12-19T22:59:40.804+0800 7fc3697e4700 -1 osd.3 21 set_numa_affinity unable to identify public
root@ceph-node2:~# ps -ef | grep osd
ceph       16284       1  0 23:00 ?        00:00:05 /usr/bin/ceph-osd -f --cluster ceph --id 4 --setuser ceph --setgroup ceph
ceph       18051       1  0 23:00 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 5 --setuser ceph --setgroup ceph
ceph       19813       1  0 23:00 ?        00:00:05 /usr/bin/ceph-osd -f --cluster ceph --id 6 --setuser ceph --setgroup ceph
ceph       21589       1  0 23:01 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 7 --setuser ceph --setgroup ceph
root       22175    1480  0 23:17 pts/0    00:00:00 grep --color=auto osd
root@ceph-node2:~# systemctl enable ceph-osd@4 ceph-osd@5 ceph-osd@6 ceph-osd@7
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@4.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@5.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@6.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@7.service → /lib/systemd/system/ceph-osd@.service.
root@ceph-node3:~# ps -ef | grep osd
ceph       16105       1  0 23:01 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 8 --setuser ceph --setgroup ceph
ceph       17869       1  0 23:01 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 9 --setuser ceph --setgroup ceph
ceph       19634       1  0 23:02 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 10 --setuser ceph --setgroup ceph
ceph       21399       1  0 23:02 ?        00:00:05 /usr/bin/ceph-osd -f --cluster ceph --id 11 --setuser ceph --setgroup ceph
root       21951    1325  0 23:20 pts/0    00:00:00 grep --color=auto osd
root@ceph-node3:~# systemctl enable ceph-osd@8 ceph-osd@9 ceph-osd@10 ceph-osd@11
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@8.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@9.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@10.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@11.service → /lib/systemd/system/ceph-osd@.service.
root@ceph-node4:~# ps -ef | grep osd
ceph       16060       1  0 23:02 ?        00:00:07 /usr/bin/ceph-osd -f --cluster ceph --id 12 --setuser ceph --setgroup ceph
ceph       17831       1  0 23:03 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 13 --setuser ceph --setgroup ceph
ceph       19592       1  0 23:03 ?        00:00:06 /usr/bin/ceph-osd -f --cluster ceph --id 14 --setuser ceph --setgroup ceph
ceph       21375       1  0 23:03 ?        00:00:05 /usr/bin/ceph-osd -f --cluster ceph --id 15 --setuser ceph --setgroup ceph
root       21932    1333  0 23:21 pts/0    00:00:00 grep --color=auto osd
root@ceph-node4:~# systemctl enable ceph-osd@12 ceph-osd@13 ceph-osd@14 ceph-osd@15
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@12.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@13.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@14.service → /lib/systemd/system/ceph-osd@.service.
Created symlink /etc/systemd/system/ceph-osd.target.wants/ceph-osd@15.service → /lib/systemd/system/ceph-osd@.service.
```
## 2.16 验证集群
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ ceph -s
  cluster:
    id:     97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-mon1,ceph-mon2,ceph-mon3 (age 63m)
    mgr: ceph-mgr1(active, since 48m), standbys: ceph-mgr2
    osd: 16 osds: 16 up (since 18m), 16 in (since 18m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   110 MiB used, 64 TiB / 64 TiB avail
    pgs:     1 active+clean

cephadmin@ceph-deploy:~/ceph-cluster$ ceph health
HEALTH_OK
```

# 3. 梳理块存储、文件存储及对象存储的使用场景
* 块存储: 块存储在使用的时候需要格式化为指定的文件系统，然后挂载使用，其对操作系统的兼容性 相对比较好(可以格式化为操作系统支持的文件系统)，挂载的时候通常是每个服务单独分配 独立的块存储，即各服务的块存储是独立且不共享使用的，如 Redis 的 master 和 slave 的 块存储是独立的、zookeeper 各节点的快存储是独立的、MySQL 的 master 和 slave 的块存 储是独立的、也可以用于私有云与公有云的虚拟机的系统盘和云盘等场景，此类场景适合使 用块存储。
  * 需要格式化，将文件直接保存到磁盘上。

* cephFS:对于需要在多个主机实现数据共享的场景，比如多个 nginx 读取由多个 tomcat 写入到存储 的数据，可以使用 ceph FS。
  * 提供数据存储的接口，是由操作系统针对块存储的应用，即由操作系统提供存储 接口，应用程序通过调用操作系统将文件保存到块存储进行持久化。

* 对象存储:而对于数据不会经常变化、删除和修改的场景，如短视频、APP 下载等，可以使用对象存储。
  * 也称为基于对象的存储，其中的文件被拆分成多个部分并散布在多个存储服务器， 在对象存储中，数据会被分解为称为“对象”的离散单元，并保存在单个存储库中，而不是 作为文件夹中的文件或服务器上的块来保存，对象存储需要一个简单的 HTTP 应用编程接 口 (API)，以供大多数客户端(各种语言)使用。
# 4. 基于ceph块存储实现块设备挂载及使用
## 4.1 创建 RBD
```bash
##创建存储池,指定 pg 和 pgp 的数量，pgp 是对存在 于 pg 的数据进行组合存储，pgp 通常等于 pg 的值
cephadmin@ceph-deploy:~/ceph-cluster$ ceph osd pool create myrbd1 64 64
pool 'myrbd1' created
cephadmin@ceph-deploy:~/ceph-cluster$ ceph osd pool --help
##对存储池启用 RBD 功能
cephadmin@ceph-deploy:~/ceph-cluster$ ceph osd pool application enable myrbd1 rbd
enabled application 'rbd' on pool 'myrbd1'
cephadmin@ceph-deploy:~/ceph-cluster$ rbd -h
##通过 RBD 命令对存储池初始化
cephadmin@ceph-deploy:~/ceph-cluster$ rbd pool init -p myrbd1
```
## 4.2 创建并验证img
```bash
## 创建一个名为 myimg1 的映像
cephadmin@ceph-deploy:~/ceph-cluster$ rbd create myimg1 --size 5G --pool myrbd1
## 创建一个名为 myimg2 的映像，并指定特性
rbd create myimg2 --size 3G --pool myrbd1 --image-format 2 --image-feature layering
#列出指定的 pool 中所有的 img
cephadmin@ceph-deploy:~/ceph-cluster$ rbd ls --pool myrbd1
myimg1
myimg2
## 查看指定 rdb 的信息
cephadmin@ceph-deploy:~/ceph-cluster$ rbd --image myimg1 --pool myrbd1 info
rbd image 'myimg1':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 12d58c7d231a
	block_name_prefix: rbd_data.12d58c7d231a
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	op_features:
	flags:
	create_timestamp: Tue Dec 20 10:46:31 2022
	access_timestamp: Tue Dec 20 10:46:31 2022
	modify_timestamp: Tue Dec 20 10:46:31 2022
cephadmin@ceph-deploy:~/ceph-cluster$ rbd --image myimg2 --pool myrbd1 info

rbd image 'myimg2':
	size 3 GiB in 768 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 12db736f9f53
	block_name_prefix: rbd_data.12db736f9f53
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: Tue Dec 20 10:47:58 2022
	access_timestamp: Tue Dec 20 10:47:58 2022
	modify_timestamp: Tue Dec 20 10:47:58 2022
```
## 4.3 客户端使用块存储
```bash
## 当前 ceph 状态
cephadmin@ceph-deploy:~/ceph-cluster$ ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  116 MiB   116 MiB          0
TOTAL  64 TiB  64 TiB  116 MiB   116 MiB          0

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS    USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0     0 B      0     20 TiB
myrbd1                  2   64   405 B        7  48 KiB      0     20 TiB
root@ceph-client1:~# apt install -y apt-transport-https ca-certificates curl software-properties-common
root@ceph-client1:~# wget -q -O- 'https://mirrors.tuna.tsinghua.edu.cn/ceph/keys/release.asc' | sudo apt-key add -
OK
root@ceph-client1:~# echo "deb https://mirrors.tuna.tsinghua.edu.cn/ceph/debian-pacific bionic main" >> /etc/apt/sources.list
root@ceph-client1:~# apt update
root@ceph-client1:~# apt install ceph-common -y
##从部署服务器同步认证文件
cephadmin@ceph-deploy:~/ceph-cluster$ scp ceph.conf ceph.client.admin.keyring root@192.168.50.151:/etc/ceph/
##客户端映射 img
root@ceph-client1:~#  rbd -p myrbd1 map myimg2
/dev/rbd0
root@ceph-client1:~#  rbd -p myrbd1 map myimg1
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable myrbd1/myimg1 object-map fast-diff deep-flatten".
In some cases useful info is found in syslog - try "dmesg | tail".
rbd: map failed: (6) No such device or address
##客户端验证 RBD
root@ceph-client1:~# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  60G  0 disk
└─sda1   8:1    0  60G  0 part /
rbd0   252:0    0   3G  0 disk
root@ceph-client1:~# fdisk -l /dev/rbd0
Disk /dev/rbd0: 3 GiB, 3221225472 bytes, 6291456 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 4194304 bytes / 4194304 bytes
##客户端格式化磁盘并挂载使用
root@ceph-client1:~# mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=9, agsize=97280 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=0, rmapbt=0, reflink=0
data     =                       bsize=4096   blocks=786432, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
root@ceph-client1:~# mkdir /data
root@ceph-client1:~# mount /dev/rbd0 /data
root@ceph-client1:~# cp /etc/passwd /data/
root@ceph-client1:~# ll /data/
total 8
drwxr-xr-x  2 root root   20 Dec 20 11:04 ./
drwxr-xr-x 24 root root 4096 Dec 20 11:03 ../
-rw-r--r--  1 root root 1682 Dec 20 11:04 passwd
root@ceph-client1:~# df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  954M     0  954M   0% /dev
tmpfs          tmpfs     198M  9.3M  188M   5% /run
/dev/sda1      ext4       59G  4.2G   52G   8% /
tmpfs          tmpfs     986M     0  986M   0% /dev/shm
tmpfs          tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs          tmpfs     986M     0  986M   0% /sys/fs/cgroup
tmpfs          tmpfs     198M     0  198M   0% /run/user/0
/dev/rbd0      xfs       3.0G   36M  3.0G   2% /data
```
## 4.4 客户端验证
```bash
root@ceph-client1:~# dd if=/dev/zero of=/data/ceph-test-file bs=10MB count=30
30+0 records in
30+0 records out
300000000 bytes (300 MB, 286 MiB) copied, 1.6221 s, 185 MB/s
root@ceph-client1:~# ll -h /data/ceph-test-file
-rw-r--r-- 1 root root 287M Dec 20 11:06 /data/ceph-test-file
root@ceph-client1:~# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL      USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  1011 MiB  1011 MiB          0
TOTAL  64 TiB  64 TiB  1011 MiB  1011 MiB          0

--- POOLS ---
POOL                   ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0     20 TiB
myrbd1                  2   64  297 MiB       92  890 MiB      0     20 TiB
```
## 4.5 删除数据
```bash
root@ceph-client1:~# rm -rf /data/ceph-test-file
## 删除完成的数据只是标记为已经被删除，但是不会从块存储立即清空，因此在删除完成后使用ceph df 查看并没有回收空间
root@ceph-client1:~# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL      USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  1012 MiB  1012 MiB          0
TOTAL  64 TiB  64 TiB  1012 MiB  1012 MiB          0

--- POOLS ---
POOL                   ID  PGS   STORED  OBJECTS     USED  %USED  MAX AVAIL
device_health_metrics   1    1      0 B        0      0 B      0     20 TiB
myrbd1                  2   64  297 MiB       92  890 MiB      0     20 TiB
## 但是后期可以使用此空间，如果需要立即在系统层回收空间，需要执行以下命令
root@ceph-client1:~# fstrim -v /data
/data: 3 GiB (3206201344 bytes) trimmed
root@ceph-client1:~# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  155 MiB   155 MiB          0
TOTAL  64 TiB  64 TiB  155 MiB   155 MiB          0

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS    USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0     0 B      0     20 TiB
myrbd1                  2   64  10 MiB       19  31 MiB      0     20 TiB
##主要用于 SSD，立即触发闲置的块回收
root@ceph-client1:~#  mount -t xfs -o discard /dev/rbd0 /data/
root@ceph-client1:~# umount /data
root@ceph-client1:~#  mount -t xfs -o discard /dev/rbd0 /data/
root@ceph-client1:~# dd if=/dev/zero of=/data/ceph-test-file bs=10MB count=30
30+0 records in
30+0 records out
300000000 bytes (300 MB, 286 MiB) copied, 1.43347 s, 209 MB/s
root@ceph-client1:~# rm -rf /data/ceph-test-file
root@ceph-client1:~# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  157 MiB   157 MiB          0
TOTAL  64 TiB  64 TiB  157 MiB   157 MiB          0

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS    USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0     0 B      0     20 TiB
myrbd1                  2   64  10 MiB       20  31 MiB      0     20 TiB
```
# 5. 基于cephFS实现多主机数据共享
## 5.1 部署MDS服务
* 在指定的 ceph-mds 服务器部署 ceph-mds 服务，可以和其它服务器混用(如 ceph-mon、 ceph-mgr)
```bash
root@ceph-mgr1:~# apt install ceph-mds=16.2.10-1bionic

cephadmin@ceph-deploy:~/ceph-cluster$ ceph-deploy mds create ceph-mgr1
[ceph_deploy.conf][DEBUG ] found configuration file at: /home/cephadmin/.cephdeploy.conf
[ceph_deploy.cli][INFO  ] Invoked (2.0.1): /usr/local/bin/ceph-deploy mds create ceph-mgr1
[ceph_deploy.cli][INFO  ] ceph-deploy options:
[ceph_deploy.cli][INFO  ]  username                      : None
[ceph_deploy.cli][INFO  ]  verbose                       : False
[ceph_deploy.cli][INFO  ]  overwrite_conf                : False
[ceph_deploy.cli][INFO  ]  subcommand                    : create
[ceph_deploy.cli][INFO  ]  quiet                         : False
[ceph_deploy.cli][INFO  ]  cd_conf                       : <ceph_deploy.conf.cephdeploy.Conf instance at 0x7f5c14b6e2d0>
[ceph_deploy.cli][INFO  ]  cluster                       : ceph
[ceph_deploy.cli][INFO  ]  func                          : <function mds at 0x7f5c14fd07d0>
[ceph_deploy.cli][INFO  ]  ceph_conf                     : None
[ceph_deploy.cli][INFO  ]  mds                           : [('ceph-mgr1', 'ceph-mgr1')]
[ceph_deploy.cli][INFO  ]  default_release               : False
[ceph_deploy.mds][DEBUG ] Deploying mds, cluster ceph hosts ceph-mgr1:ceph-mgr1
[ceph-mgr1][DEBUG ] connection detected need for sudo
[ceph-mgr1][DEBUG ] connected to host: ceph-mgr1
[ceph-mgr1][DEBUG ] detect platform information from remote host
[ceph-mgr1][DEBUG ] detect machine type
[ceph_deploy.mds][INFO  ] Distro info: Ubuntu 18.04 bionic
[ceph_deploy.mds][DEBUG ] remote host will use systemd
[ceph_deploy.mds][DEBUG ] deploying mds bootstrap to ceph-mgr1
[ceph-mgr1][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
[ceph-mgr1][WARNIN] mds keyring does not exist yet, creating one
[ceph-mgr1][DEBUG ] create a keyring file
[ceph-mgr1][DEBUG ] create path if it doesn't exist
[ceph-mgr1][INFO  ] Running command: sudo ceph --cluster ceph --name client.bootstrap-mds --keyring /var/lib/ceph/bootstrap-mds/ceph.keyring auth get-or-create mds.ceph-mgr1 osd allow rwx mds allow mon allow profile mds -o /var/lib/ceph/mds/ceph-ceph-mgr1/keyring
[ceph-mgr1][INFO  ] Running command: sudo systemctl enable ceph-mds@ceph-mgr1
[ceph-mgr1][WARNIN] Created symlink /etc/systemd/system/ceph-mds.target.wants/ceph-mds@ceph-mgr1.service → /lib/systemd/system/ceph-mds@.service.
[ceph-mgr1][INFO  ] Running command: sudo systemctl start ceph-mds@ceph-mgr1
[ceph-mgr1][INFO  ] Running command: sudo systemctl enable ceph.target
```
## 5.2 验证 MDS 服务
* MDS 服务目前还无法正常使用，需要为 MDS 创建存储池用于保存 MDS 的数据
```bash
cephadmin@ceph-deploy:~/ceph-cluster$  ceph mds stat
 1 up:standby
## 当前为备用状态，需要分配 pool 才可以使用。
```
## 5.3 创建 CephFS metadata 和 data 存储池
* 使用 CephFS 之前需要事先于集群中创建一个文件系统，并为其分别指定元数据和数据相 关的存储池，如下命令将创建名为 mycephfs 的文件系统，它使用cephfs-metadata 作为 元数据存储池，使用 cephfs-data 为数据存储池:
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ ceph osd pool create cephfs-metadata 32 32
pool 'cephfs-metadata' created ## 保存 metadata 的 pool
cephadmin@ceph-deploy:~/ceph-cluster$ ceph osd pool create cephfs-data 64 64
pool 'cephfs-data' created ## 保存数据的 pool
cephadmin@ceph-deploy:~/ceph-cluster$ ceph -s
  cluster:
    id:     97a6b32d-4af8-4dde-b3f5-f5ae5db0824f
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum ceph-mon1,ceph-mon2,ceph-mon3 (age 13h)
    mgr: ceph-mgr1(active, since 12h), standbys: ceph-mgr2
    osd: 16 osds: 16 up (since 12h), 16 in (since 12h)

  data:
    pools:   4 pools, 161 pgs
    objects: 20 objects, 10 MiB
    usage:   164 MiB used, 64 TiB / 64 TiB avail
    pgs:     161 active+clean
```
## 5.4 创建 cephFS 并验证
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ ceph fs new mycephfs cephfs-metadata cephfs-data
new fs with metadata pool 3 and data pool 4
cephadmin@ceph-deploy:~/ceph-cluster$ ceph fs ls
name: mycephfs, metadata pool: cephfs-metadata, data pools: [cephfs-data ]
cephadmin@ceph-deploy:~/ceph-cluster$ ceph fs status mycephfs
mycephfs - 0 clients
========
RANK  STATE      MDS        ACTIVITY     DNS    INOS   DIRS   CAPS
 0    active  ceph-mgr1  Reqs:    0 /s    10     13     12      0
      POOL         TYPE     USED  AVAIL
cephfs-metadata  metadata  96.0k  20.2T
  cephfs-data      data       0   20.2T
MDS version: ceph version 16.2.10 (45fa1a083152e41a408d15505f594ec5f1b4fe17) pacific (stable)
```
## 5.5 验证 cepfFS 服务状态
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ ceph mds stat
mycephfs:1 {0=ceph-mgr1=up:active}
##cephfs 状态现在已经转变为活动状态
```
## 5.6 客户端挂载 cephFS
```bash
cephadmin@ceph-deploy:~/ceph-cluster$ cat ceph.client.admin.keyring
[client.admin]
	key = AQCwcqBjeQ26IxAARXz0XV+Art0ugawMcwAMxg==
	caps mds = "allow *"
	caps mgr = "allow *"
	caps mon = "allow *"
	caps osd = "allow *"
	
root@ceph-client1:~# mount -t ceph 192.168.50.105:6789:/ /mnt -o name=admin,secret=AQCwcqBjeQ26IxAARXz0XV+Art0ugawMcwAMxg==
root@ceph-client1:~# df -Th
Filesystem            Type      Size  Used Avail Use% Mounted on
udev                  devtmpfs  954M     0  954M   0% /dev
tmpfs                 tmpfs     198M  9.4M  188M   5% /run
/dev/sda1             ext4       59G  4.2G   52G   8% /
tmpfs                 tmpfs     986M     0  986M   0% /dev/shm
tmpfs                 tmpfs     5.0M     0  5.0M   0% /run/lock
tmpfs                 tmpfs     986M     0  986M   0% /sys/fs/cgroup
tmpfs                 tmpfs     198M     0  198M   0% /run/user/0
192.168.50.105:6789:/ ceph       21T     0   21T   0% /mnt
## 数据验证
root@ceph-client1:~# cp /var/log/syslog /mnt/
root@ceph-client1:~# ls /mnt/
syslog
## 测试数据写入
root@ceph-client1:~# dd if=/dev/zero of=/mnt/ceph-test-file bs=10MB count=10
10+0 records in
10+0 records out
100000000 bytes (100 MB, 95 MiB) copied, 0.0961591 s, 1.0 GB/s
root@ceph-client1:~# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  455 MiB   455 MiB          0
TOTAL  64 TiB  64 TiB  455 MiB   455 MiB          0

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS     USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0      0 B      0     20 TiB
myrbd1                  2   64  10 MiB       20   31 MiB      0     20 TiB
cephfs-metadata         3   32  20 KiB       22  144 KiB      0     20 TiB
cephfs-data             4   64  96 MiB       25  289 MiB      0     20 TiB
## 删除数据
root@ceph-client1:~# cd /mnt/
root@ceph-client1:/mnt# ls
ceph-test-file  syslog
root@ceph-client1:/mnt# rm ceph-test-file syslog  -f
root@ceph-client1:/mnt# ceph df
--- RAW STORAGE ---
CLASS    SIZE   AVAIL     USED  RAW USED  %RAW USED
hdd    64 TiB  64 TiB  167 MiB   167 MiB          0
TOTAL  64 TiB  64 TiB  167 MiB   167 MiB          0

--- POOLS ---
POOL                   ID  PGS  STORED  OBJECTS     USED  %USED  MAX AVAIL
device_health_metrics   1    1     0 B        0      0 B      0     20 TiB
myrbd1                  2   64  10 MiB       20   31 MiB      0     20 TiB
cephfs-metadata         3   32  33 KiB       23  192 KiB      0     20 TiB
cephfs-data             4   64     0 B        0      0 B      0     20 TiB
```
## 5.7 数据共享
```bash
## 客户端2上也挂载 cephFS
root@ceph-client2:~# mount -t ceph 192.168.50.105:6789:/ /mnt -o name=admin,secret=AQCwcqBjeQ26IxAARXz0XV+Art0ugawMcwAMxg==
root@ceph-client2:~# df -h
Filesystem             Size  Used Avail Use% Mounted on
udev                   954M     0  954M   0% /dev
tmpfs                  198M  9.1M  188M   5% /run
/dev/sda1               59G  4.2G   52G   8% /
tmpfs                  986M     0  986M   0% /dev/shm
tmpfs                  5.0M     0  5.0M   0% /run/lock
tmpfs                  986M     0  986M   0% /sys/fs/cgroup
tmpfs                  198M     0  198M   0% /run/user/0
192.168.50.105:6789:/   21T     0   21T   0% /mnt
## 客户端1上创建一个html文件
root@ceph-client1:/mnt# echo "<h1> CephFS share test! </h1>" >> /mnt/index.html
## 客户端2上可以读取到html文件
root@ceph-client2:~# cat /mnt/index.html
<h1> CephFS share test! </h1>
```