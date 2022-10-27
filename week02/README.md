# 1. 基于dockerfile，实现分层构建的nginx业务镜像 
## 1.1 制作ubuntu系统基础镜像
选用ubuntu-22.10作为基础镜像，安装需要使用的基本软件
```
root@docker-server1:~# cd /opt/
root@docker-server1:/opt# mkdir -p nginx-img/system-base
root@docker-server1:/opt# cd nginx-img/system-base/
root@docker-server1:/opt/nginx-img/system-base# docker build -t harbor.yanggc.cn/system/ubuntu-base:22.10 .

```
制作镜像的Dockerfile:
```
FROM ubuntu:22.10
LABEL MAINTAINER="ygc"
RUN apt update && apt install -y \
iproute2 \
ntpdate  \
tcpdump \
telnet \
traceroute \
nfs-kernel-server \
nfs-common \
lrzsz \
tree \
openssl \
libssl-dev \
libpcre3 \
libpcre3-dev \
zlib1g-dev \
ntpdate \
tcpdump \
telnet \
traceroute \
gcc \
openssh-server \
lrzsz \
tree \
openssl \
libssl-dev \
libpcre3 \
libpcre3-dev \
zlib1g-dev \
ntpdate \
tcpdump \
telnet \
traceroute \
iotop \
unzip \
zip \
make && \
apt clean
```

## 1.2 各种 Namespace 的作用
### 1.2.1 Mount Namespace
Mount Namespace 是 Linux 内核实现的第一个 Namespace，从内核 2.4.19 版本开始加入。它可以用来隔离不同的进程或者进程组看到的挂载点。通俗地说，就是可以实现在不同的进程中看到不同的挂载目录。使用 Mount Namespace 可以实现容器内只能看到自己的挂载信息，在容器内的挂载操作不会影响主机的挂载目录。
下面我们通过一个实例来演示下 Mount Namespace。在演示之前，我们先来认识一个命令行工具 unshare。unshare 是 util-linux 工具包中的一个工具，Ubuntu 22.04 系统默认已经集成了该工具，使用 unshare 命令可以实现创建并访问不同类型的 Namespace。

首先我们使用以下命令创建一个 bash 进程并且新建一个 Mount Namespace：
```bash
#新建一个 Mount Namespace
root@ubuntu:~# unshare --mount --fork /bin/bash
```
执行完上述命令后，这时我们已经在主机上创建了一个新的 Mount Namespace，并且当前命令行窗口加入了新创建的 Mount Namespace。下面我通过一个例子来验证下，在独立的 Mount Namespace 内创建挂载目录是不影响主机的挂载目录的。

然后在 /tmp 目录下创建一个目录，创建好目录后使用 mount 命令挂载一个 tmpfs 类型的目录：
```bash
root@ubuntu:~# mkdir /tmp/tmpfs
root@ubuntu:~# mount -t tmpfs -o size=20m tmpfs /tmp/tmpfs
```
然后使用 df 命令查看一下已挂载的目录信息：
```
root@ubuntu:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        39G  1.5G   38G   4% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           393M  976K  392M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           393M  4.0K  393M   1% /run/user/1000
vagrant         466G  107G  360G  23% /vagrant
tmpfs            20M     0   20M   0% /tmp/tmpfs
```
可以看到 /tmp/tmpfs 目录已经被正确挂载。为了验证主机上并没有挂载此目录，我们新打开一个命令行窗口，同样执行 df 命令查看主机的挂载信息：
```
root@ubuntu:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           393M  984K  392M   1% /run
/dev/sda1        39G  1.5G   38G   4% /
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           393M  4.0K  393M   1% /run/user/1000
```
通过上面输出可以看到主机上并没有挂载 /tmp/tmpfs，可见我们独立的 Mount Namespace 中执行 mount 操作并不会影响主机。  
为了进一步验证我们的想法，我们继续在当前命令行窗口查看一下当前进程的 Namespace 信息，命令如下：
```
root@ubuntu:~# ls -l /proc/self/ns/
total 0
lrwxrwxrwx 1 root root 0 Oct 19 14:38 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 mnt -> 'mnt:[4026532203]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Oct 19 14:38 uts -> 'uts:[4026531838]'
```
然后新打开一个命令行窗口，使用相同的命令查看一下主机上的 Namespace 信息：
```
root@ubuntu:~# ls -l /proc/self/ns/
total 0
lrwxrwxrwx 1 root root 0 Oct 19 14:43 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 net -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Oct 19 14:43 uts -> 'uts:[4026531838]'
```
通过对比两次命令的输出结果，我们可以看到，除了 Mount Namespace 的 ID 值不一样外，其他Namespace 的 ID 值均一致。

通过以上结果我们可以得出结论，使用 unshare 命令可以新建 Mount Namespace，并且在新建的 Mount Namespace 内 mount 是和外部完全隔离的。

### 1.2.2 PID Namespace
PID Namespace 的作用是用来隔离进程。在不同的 PID Namespace 中，进程可以拥有相同的 PID 号，利用 PID Namespace 可以实现每个容器的主进程为 1 号进程，而容器内的进程在主机上却拥有不同的PID。例如一个进程在主机上 PID 为 122，使用 PID Namespace 可以实现该进程在容器内看到的 PID 为 1。

下面我们通过一个实例来演示下 PID Namespace的作用。首先我们使用以下命令创建一个 bash 进程，并且新建一个 PID Namespace：
```bash
root@ubuntu:~# unshare --pid --fork --mount-proc /bin/bash
```

执行完上述命令后，我们在主机上创建了一个新的 PID Namespace，并且当前命令行窗口加入了新创建的 PID Namespace。在当前的命令行窗口使用 ps aux 命令查看一下进程信息：
```
root@ubuntu:~# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1   7632  4216 pts/1    S    13:45   0:00 /bin/bash
root           8  0.0  0.0  10068  1584 pts/1    R+   13:47   0:00 ps aux
```
通过上述命令输出结果可以看到当前 Namespace 下 bash 为 1 号进程，而且我们也看不到主机上的其他进程信息。

### 1.2.3 UTS Namespace
UTS Namespace 主要是用来隔离主机名的，它允许每个 UTS Namespace 拥有一个独立的主机名。例如我们的主机名称为 docker，使用 UTS Namespace 可以实现在容器内的主机名称为 lagoudocker 或者其他任意自定义主机名。  
同样我们通过一个实例来验证下 UTS Namespace 的作用，首先我们使用 unshare 命令来创建一个 UTS Namespace：
```bash
root@ubuntu:~# unshare --uts --fork /bin/bash
```
创建好 UTS Namespace 后，当前命令行窗口已经处于一个独立的 UTS Namespace 中，下面我们使用 hostname 命令（hostname 可以用来查看主机名称）设置一下主机名：
```bash
root@ubuntu:~# hostname -b ygc
#查看更改后主机名
root@ubuntu:~# hostname
ygc
```
通过上面命令的输出，我们可以看到当前UTS Namespace 内的主机名已经被修改为 lagoudocker。然后我们新打开一个命令行窗口，使用相同的命令查看一下主机的 hostname：
```bash
root@ubuntu:~# hostname
ubuntu
```
可以看到主机的名称仍然为 docker-demo，并没有被修改。由此，可以验证 UTS Namespace 可以用来隔离主机名。

### 1.2.3 IPC Namespace
IPC Namespace 主要是用来隔离进程间通信的。例如 PID Namespace 和 IPC Namespace 一起使用可以实现同一 IPC Namespace 内的进程彼此可以通信，不同 IPC Namespace 的进程却不能通信。

同样我们通过一个实例来验证下IPC Namespace的作用，首先我们使用 unshare 命令来创建一个 IPC Namespace：
```bash
root@ubuntu:~# unshare --ipc --fork /bin/bash
```
下面我们需要借助两个命令来实现对 IPC Namespace 的验证。

* ipcs -q 命令：用来查看系统间通信队列列表。
* ipcmk -Q 命令：用来创建系统间通信队列。
我们首先使用 ipcs -q 命令查看一下当前 IPC Namespace 下的系统通信队列列表：
```bash
root@ubuntu:~# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```
由上可以看到当前无任何系统通信队列，然后我们使用 ipcmk -Q 命令创建一个系统通信队列：
```bash
root@ubuntu:~# ipcmk -Q
Message queue id: 0
```
再次使用 ipcs -q 命令查看当前 IPC Namespace 下的系统通信队列列表：
```bash
root@ubuntu:~# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x16cd9575 0          root       644        0            0
```
可以看到我们已经成功创建了一个系统通信队列。然后我们新打开一个命令行窗口，使用ipcs -q 命令查看一下主机的系统通信队列：
```bash
root@ubuntu:~# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```
通过上面的实验，可以发现，在单独的 IPC Namespace 内创建的系统通信队列在主机上无法看到。即 IPC Namespace 实现了系统通信队列的隔离。

### 1.2.4 User Namespace
User Namespace 主要是用来隔离用户和用户组的。一个比较典型的应用场景就是在主机上以非 root 用户运行的进程可以在一个单独的 User Namespace 中映射成 root 用户。使用 User Namespace 可以实现进程在容器内拥有 root 权限，而在主机上却只是普通用户。User Namesapce 的创建是可以不使用 root 权限的，下面我们以普通用户的身份创建一个 User Namespace，命令如下：
```bash
# 创建并切换到普通用户
root@ubuntu:~# su - ygc
ygc@ubuntu:~$ id
uid=1002(ygc) gid=1002(ygc) groups=1002(ygc)
# 创建 User Namespace
ygc@ubuntu:~$ unshare --user -r /bin/bash
```
然后执行 id 命令查看一下当前的用户信息：
```bash
root@ubuntu:~# id
uid=0(root) gid=0(root) groups=0(root)
```
通过上面的输出可以看到我们在新的 User Namespace 内已经是 root 用户了。下面我们使用只有主机 root 用户才可以执行的 reboot 命令来验证一下，在当前命令行窗口执行 reboot 命令：
```bash
root@ubuntu:~# reboot
Failed to connect to bus: Operation not permitted (consider using --machine=<user>@.host --user to connect to bus of other user)
Failed to open initctl fifo: Permission denied
Failed to talk to init daemon.
```
查看当前 User Namespace 所在的进程：
```bash
root@ubuntu:~# echo $$
1667
```
在宿主机上查看该进程对应的用户，可以看到实际上在宿主机上是 chengzw 用户：
```bash
root@ubuntu:~# ps -ef | grep 1667
ygc         1667    1648  0 03:37 pts/1    00:00:00 /bin/bash
root        1760    1750  0 03:41 pts/3    00:00:00 grep --color=auto 1667
```
可以看到，我们在新创建的 User Namespace 内虽然是 root 用户，但是并没有权限执行 reboot 命令。这说明在隔离的 User Namespace 中，并不能获取到主机的 root 权限，也就是说 User Namespace 实现了用户和用户组的隔离。

### 1.2.5 Net Namespace
Net Namespace 是用来隔离网络设备、IP 地址和端口等信息的。Net Namespace 可以让每个进程拥有自己独立的 IP 地址，端口和网卡信息。例如主机 IP 地址为 192.168.34.3 ，容器内可以设置独立的 IP 地址为 192.168.100.2。

同样用实例验证，我们首先使用 ip addr 命令查看一下主机上的网络信息：
```bash
root@ubuntu:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:e1:b2:13:a8:c4 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 metric 100 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 85350sec preferred_lft 85350sec
    inet6 fe80::e1:b2ff:fe13:a8c4/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:04:a7:44 brd ff:ff:ff:ff:ff:ff
    inet 192.168.34.3/24 brd 192.168.34.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe04:a744/64 scope link
       valid_lft forever preferred_lft forever
```
然后我们使用以下命令创建一个 Net Namespace：
```bash
root@ubuntu:~# unshare --net --fork /bin/bash
```
同样的我们使用 ip addr 命令查看一下网络信息：
```bash
root@ubuntu:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
可以看到，宿主机上有 lo、enp0s8 等网络设备，而我们新建的 Net Namespace 内则与主机上的网络设备不同。我们可以在 Net Namespace 中新增网卡并配置 IP：
```bash
root@ubuntu:~# ip link add eth0 type dummy
root@ubuntu:~# ip addr add 192.168.100.2/24 dev eth0
root@ubuntu:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 32:0a:9f:57:ee:44 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.2/24 scope global eth0
       valid_lft forever preferred_lft forever
```
### 1.2.6 Time Namespace
 Time Namespace提供时间隔离能力，系统调用参数 CLONE_NEWTIME，在 Linux 5.6 内核版本开始加入 linux 系统。

同样用实例验证，我们使用uptime查看系统时间：
```bash
root@ubuntu:~# uptime -s
2022-10-21 07:04:31
```
然后我们使用以下命令创建一个 Time Namespace：
```
root@ubuntu:~# unshare -T --boottime 1000000000 --fork /bin/bash
root@ubuntu:~# uptime -s
1991-02-12 05:17:51
```
可以看到新建的Time Namespace时间和宿主机上的时间是不同的。

# 2. 使用 apt/yum/ 二进制安装指定版本的 Docker
## 2.1 apt安装docker
```bash
# step 1: 安装必要的一些系统工具
root@ubuntu:~# apt update
root@ubuntu:~# apt -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
root@ubuntu:~# curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
root@ubuntu:~# add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
root@ubuntu:~# apt update
# Step 5: 查找Docker-CE的版本:
root@ubuntu:~# apt-cache madison docker-ce
 docker-ce | 5:20.10.18~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
 docker-ce | 5:20.10.17~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
 docker-ce | 5:20.10.16~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
 docker-ce | 5:20.10.15~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
 docker-ce | 5:20.10.14~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
 docker-ce | 5:20.10.13~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
# Step 5: 安装指定版本的Docker-CE: (VERSION例如上面的5:20.10.18~3-0~ubuntu-jammy)
apt install docker-ce=5:20.10.18~3-0~ubuntu-jammy
# Step 6:验证安装
root@ubuntu:~# docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Docker Buildx (Docker Inc., v0.9.1-docker)
  scan: Docker Scan (Docker Inc., v0.17.0)

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 20.10.18
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runtime.v1.linux runc io.containerd.runc.v2
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
 runc version: v1.1.4-0-g5fd4c4d
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: default
  cgroupns
 Kernel Version: 5.15.0-50-generic
 Operating System: Ubuntu 22.04.1 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.832GiB
 Name: ubuntu
 ID: 67XU:F7UN:UIUU:3URO:BIKK:QVHD:HE6A:G6OX:YMIA:BDC4:6NVT:7U2K
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
# 输出了client和server端信息，证明安装成功
```
## 2.1 二进制安装docker
```bash
# Step 1: 拷贝二进制文件到虚拟机
#vagrant scp docker-20.10.18-binary-install.tar.gz default:~/
# Step 2: 解压二进制文件
root@ubuntu:~# tar -xzvf docker-20.10.18-binary-install.tar.gz
# Step 3: 执行安装脚本
root@ubuntu:~/docker-20.10.18-binary-install# ./docker-install.sh
当前系统是Ubuntu 22.04.1 LTS \n \l,即将开始系统初始化、配置docker-compose与安装docker
docker/
docker/docker
docker/containerd
docker/runc
docker/docker-proxy
docker/dockerd
docker/ctr
docker/docker-init
docker/containerd-shim
docker/containerd-shim-runc-v2
正在启动docker server并设置为开机自启动!
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /lib/systemd/system/containerd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
docker server安装完成,欢迎进入docker世界!
# Step 4: 验证安装
root@ubuntu:~/docker-20.10.18-binary-install# ./docker-install.sh
当前系统是Ubuntu 22.04.1 LTS \n \l,即将开始系统初始化、配置docker-compose与安装docker
docker/
docker/docker
docker/containerd
docker/runc
docker/docker-proxy
docker/dockerd
docker/ctr
docker/docker-init
docker/containerd-shim
docker/containerd-shim-runc-v2
正在启动docker server并设置为开机自启动!
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /lib/systemd/system/containerd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
docker server安装完成,欢迎进入docker世界!
root@ubuntu:~/docker-20.10.18-binary-install# docker info
Client:
 Context:    default
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 20.10.18
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 io.containerd.runtime.v1.linux runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
 runc version: v1.1.4-0-g5fd4c4d1
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: default
  cgroupns
 Kernel Version: 5.15.0-50-generic
 Operating System: Ubuntu 22.04.1 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.832GiB
 Name: ubuntu
 ID: ORNK:QRPU:L3RH:JGJE:5GGK:ZM4W:TVP6:KRED:L76P:D3UL:JQV2:Z2QL
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  172.31.7.105
  harbor.magedu.com
  harbor.myserver.com
  127.0.0.0/8
 Registry Mirrors:
  https://9916w1ow.mirror.aliyuncs.com/
 Live Restore Enabled: false
 Product License: Community Engine
 # 输出了client和server端信息，证明安装成功
```

# 3.熟练使用 Docker 数据卷
## 3.1 Docker 数据卷概念
* Docker的镜像是分层设计的，镜像层是只读的，通过镜像启动的容器添加了一层可读写的文件系统，用户写入的数据都保存在这一层当中。
* 如果要将写入到容器的数据永久保存，则需要将容器中的数据保存到宿主机的指定目录，目前Docker的数据类型分为两种:
* * 一是数据卷(data volume)，数据卷实际上就是宿主机上的目录或者是文件，可以被直接mount到容器当中作为容器的本地文件系统使 用，实际生产环境中，需要针对不同类型的服务、不同类型的数据存储要求做相应的规划，最终保证服务的可扩展性、稳定性以及数据的 安全性。基于数据卷通过将宿主机的文件或目录挂载到容器的指定目录，当容器中的挂载目录产生的新数据即可间接的保存到宿主机以实 现持久化的目的.
* * 二是数据卷容器(Data volume container), 数据卷容器是将宿主机的目录挂载至一个专用的数据卷容器，然后让其他容器通过数据卷容器 继承挂载的目录或文件，以实现将新产生的数据持久化到宿主机的目的。
## 3.2 数据卷
### 3.2.1 创建数据卷
```bash
# Step 1: 创建数据卷
root@ubuntu:~# docker volume create nginx-data
nginx-data
# 查看数据卷
root@ubuntu:~# docker volume ls
DRIVER    VOLUME NAME
local     nginx-data
# Step 2: 使用建数据卷创建nginx容器
root@ubuntu:~# docker run -it -d -p 80:80 -v nginx-data:/data nginx:1.23.2
# Step 3: 测试创建的nginx容器是否运行
root@ubuntu:~# curl localhost
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
# Step 4: 登录nginx容器，写入一条数据到数据卷的index.html文件中
root@ubuntu:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                               NAMES
ef6be191b34b   nginx:1.23.2   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp   gracious_maxwell
root@ubuntu:~# docker exec -it ef6be191b34b bash
root@ef6be191b34b:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          39G  2.1G   37G   6% /
tmpfs            64M     0   64M   0% /dev
shm              64M     0   64M   0% /dev/shm
/dev/sda1        39G  2.1G   37G   6% /data
tmpfs           2.0G     0  2.0G   0% /proc/acpi
tmpfs           2.0G     0  2.0G   0% /proc/scsi
tmpfs           2.0G     0  2.0G   0% /sys/firmware
root@ef6be191b34b:/# echo "nginx date volume test" > /data/index.html
root@ef6be191b34b:/# exit
exit
# Step 5: 在宿主机查看文件内容，和容器内写入的数据是一样的
root@ubuntu:~#  ls /var/lib/docker/volumes/nginx-data/_data/index.html
/var/lib/docker/volumes/nginx-data/_data/index.html
root@ubuntu:~# cat /var/lib/docker/volumes/nginx-data/_data/index.html
nginx date volume test
```
### 3.2.2 数据目录挂载
以数据卷的方式，将自定义的宿主机目录或文件提供给容器使用，比如容器可以直接挂载宿主机本地的数据目录(如mysql容器的数据持久化)、 配置文件(如nginx的配置文件)、静态文件(如web服务的图片或js文件)等，只需要在创建容器的时候指定挂载即可。
```bash
# Step 1: 创建一个目录，生成一个index.html文件
root@ubuntu:~# mkdir /data/testapp -p
root@ubuntu:~# echo "testapp web page" > /data/testapp/index.html
root@ubuntu:~# cat /data/testapp/index.html
testapp web page
# Step 2: 启动两个测试容器，web1容器和web2容器，分别测试能否访问到宿主机的数据，注意使用-v参数，将宿主机目录映射到容器内部，web2的ro 表示在容器内对该目录只读，默认的权限是可读写的
root@ubuntu:~# docker run -d --name web1 -v /data/testapp:/usr/share/nginx/html -p 80:80 nginx:1.23.2
e2b502cc83a13639842c1588c3cce10b71c7790bbea4f17a3c53736df6c91bbd
root@ubuntu:~# curl localhost
testapp web page
root@ubuntu:~# docker run -d --name web2 -v /data/testapp:/usr/share/nginx/html:ro -p 81:80 nginx:1.23.2
82fa4ee89b267bccaae731d669d575520c74fd66af8b3dbb86068aa3e09bc7a4
root@ubuntu:~# curl localhost:81
testapp web page
```
### 3.2.3 数据目录及配置多卷挂载
* ngin多卷挂载
```bash
# 创建nginx配置文件目录
# mkdir /data/nginx/conf -p
#拷贝 web1 上的default.conf 配置文件
# docker cp web1:/etc/nginx/conf.d/default.conf /data/nginx/conf/
# default.conf 添加一段配置
    location ~ ^/auok {
            default_type text/plain;
            return 200 IAMOK;
    }
# 启动一个新的web3，挂载nginx配置目录和ning首页文件目录
root@ubuntu:~# docker run -d --name web3 -v /data/testapp:/usr/share/nginx/html -v /data/nginx/conf:/etc/nginx/conf.d:ro -p 83:80 nginx:1.23.2
# 验证
root@ubuntu:~# curl localhost:83
testapp web page
root@ubuntu:~# curl localhost:83/auok
IAMOK
```
* mysql容器
```bash
# Step 1: 下载mysql镜像
root@ubuntu:~# docker pull mysql:5.7.40
# Step 2: 启动mysql容器，并挂载/data/mysql目录
root@ubuntu:~# docker run -it -d -p 3306:3306 -v /data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7.40
7d225a833d0aacce619a8b0b752578be8fa356a405ff57eb38f263cb46aef4ad
# Step 3: 测试登录mysql
root@ubuntu:~# mysql -uroot -p123456 -h172.17.0.1
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
### 3.2.4 删除容器
* 创建容器的时候指定参数会删除/var/lib/docker/containers/的容器数据目录，但是不会删除数据卷的内容，如下
```bash
root@ubuntu:~# docker rm -f web3
web3
root@ubuntu:~# ls /data/testapp/index.html
/data/testapp/index.html
root@ubuntu:~# ls /data/nginx/conf/default.conf
/data/nginx/conf/default.conf
# 载的数据卷不会被删除
```
* 数据卷的特点及使用:
  * 数据卷是宿主机的目录或者文件，并且可以在多个容器之间共同使用。
  * 在宿主机对数据卷更改数据后会在所有容器里面会立即更新。
  * 数据卷的数据可以持久保存，即使删除使用使用该容器卷的容器也不影响。
  * 在容器里面的写入数据不会影响到镜像本身。
* 数据卷使用场景:
  * 容器数据持久化(mysql数据、nginx日志等类型)  静态web页面挂载
  * 应用配置文件挂载
  * 多容器间的目录或文件共享

### 3.2.5 数据卷容器
* 数据卷容器功能是可以让数据在多个docker容器之间共享，即先要创建一个后台运行的A容器作为Server，之后创建的B容器、C容 器等都可以同时访问A容器的内容，因此数据卷容器用于为其它容器提供卷的挂载继承服务，数据卷为其它容器提供数据读写服务， A容器称为server端、其它容器成为client端
```bash
# Step 1: 创建一个卷容器server
root@ubuntu:~# docker run -d --name volume-server -v /data/testapp:/usr/share/nginx/html -v /data/nginx/conf:/etc/nginx/conf.d:ro ibmcom/pause:3.1
# Step 2: 启动client1和client2
root@ubuntu:~# docker run -d --name web1 --volumes-from volume-server -p80:80 nginx:1.23.2
ffd6c18400ee21523b3cd8c385eb3121ee1f3b681cb20fdde1a1bd26662e11e3
root@ubuntu:~# docker run -d --name web2 --volumes-from volume-server -p81:80 nginx:1.23.2
96097ae996f9835b130431501af4e8ce166ec91671ef2de871924f862c4b8ffc
# Step 3: 测试client1和client2引用卷容器server挂载的数据库是否正常
root@ubuntu:~# curl localhost
testapp web page
root@ubuntu:~# curl localhost/auok
IAMOK
root@ubuntu:~# curl localhost:81
testapp web page
root@ubuntu:~# curl localhost:81/auok
IAMOK
```
* 特点:
  * 适用于同类服务的数据卷共享
  * client会继承卷server挂载和挂载权限
  * 停止卷server，也不影响已经运行的容器、甚至也不影响新建容器 
  * 删除卷server，不影响已经运行的容器，但是不能新建容器

# 4. 熟练使用 Docker 的 bridge 和 container 模式网络
## 4.1 docker网络
*  Docker服务安装完成之后，默认在每个宿主机会生成一个名称为docker0的网卡其IP地址都是172.17.0.1/16，并且会生成三种不能类型的网 络，如下：
```bash
root@ubuntu:~# ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:d6ff:fe32:27db  prefixlen 64  scopeid 0x20<link>
        ether 02:42:d6:32:27:db  txqueuelen 0  (Ethernet)
        RX packets 119  bytes 16741 (16.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 161  bytes 14515 (14.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
* 另外会额外创建三个默认网络，用于不同的使用场景:
```bash
root@ubuntu:~# docker network list
NETWORK ID     NAME      DRIVER    SCOPE
380550f19502   bridge    bridge    local
0a0bc8b22bed   host      host      local
3921a9a0731f   none      null      local
```
### 4.1.1 bridge模式
* docker的默认模式即不指定任何模式就是bridge模式，也是目前使用比较多的网络模式，此模式创建的容器会为每一个容器分配自己的网络 IP等信息，并将容器连接到一个虚拟网桥与外界通信。
* #docker network inspect bridge
```bash
root@ubuntu:~# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "380550f195024a8f038509a0f64ad350da1370eec6215fe689d0f760ec4c344b",
        "Created": "2022-10-22T06:46:36.951680929Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```
* 创建一个bridge模式的容器
```bash
root@ubuntu:~# docker run -it -d --name nginx-web1-bridge-test-container -p 80:80 --net=bridge nginx:1.23.2
2c16fe5be2b294d039bad7f2934c4c5273e799e2108817580410cc7b14b606b3
```
### 4.1.2 host模式
* host模式就是直接使用宿主机的IP地址，创建的容器如果指定了使用host模式，那么新创建的容器不会创建自己的虚拟网卡，而是直接使用宿 主机的网卡和IP地址，因此在容器里面查看到的IP信息就是宿主机的信息，访问容器的时候直接使用宿主机IP+容器端口即可，而且某一个端 口一旦被其中一个容器占用那么其它容器就不能再监听此端口，不过容器的其它资源比如容器的文件系统、容器系统进程等还是基于 namespace相互隔离
* 查看host模式网络信息
```bash
root@ubuntu:~# docker network inspect host
[
    {
        "Name": "host",
        "Id": "0a0bc8b22bedad4b851905613067934e7974eaf30709b0bab9f43b26e9497f4b",
        "Created": "2022-10-22T06:38:25.260386298Z",
        "Scope": "local",
        "Driver": "host",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": []
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
* 此模式的网络性能最高，但是各容器之间端口不能相同，适用于运行容器端口比较固定的业务。
* 创建一个host网络模式的容器
```bash
root@ubuntu:~# docker run -it -d --name nginx-web1-host-test-container -p 80:80 --net=host nginx:1.23.2
WARNING: Published ports are discarded when using host network mode
d0a9ca667419180296a6e0164a4e2f3fde511cac864851bf13098bba96faa450
root@ubuntu:~# docker exec -it nginx-web1-host-test-container bash
root@ubuntu:/# apt update && apt install net-tools
root@ubuntu:/# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:d6ff:fe32:27db  prefixlen 64  scopeid 0x20<link>
        ether 02:42:d6:32:27:db  txqueuelen 0  (Ethernet)
        RX packets 119  bytes 16741 (16.3 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 163  bytes 14735 (14.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::e1:b2ff:fe13:a8c4  prefixlen 64  scopeid 0x20<link>
        ether 02:e1:b2:13:a8:c4  txqueuelen 1000  (Ethernet)
        RX packets 258688  bytes 364405445 (347.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 37752  bytes 2749881 (2.6 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.34.3  netmask 255.255.255.0  broadcast 192.168.34.255
        inet6 fe80::a00:27ff:fe89:dda2  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:89:dd:a2  txqueuelen 1000  (Ethernet)
        RX packets 43  bytes 4197 (4.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 53  bytes 5462 (5.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 830  bytes 68974 (67.3 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 830  bytes 68974 (67.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@ubuntu:/# netstat -tanlp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1/nginx: master pro
tcp        0      0 127.0.0.1:42037         0.0.0.0:*               LISTEN      -
tcp        0      0 10.0.2.15:22            10.0.2.2:49299          ESTABLISHED -
tcp6       0      0 :::22                   :::*                    LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      1/nginx: master pro
```
### 4.1.3 none模式
* none模式就是无IP模式，在使用none 模式后，docker 容器不会进行任何网络配置，其没有网卡、没有IP也没有路由，因此默认无法与外界通 信，需要手动添加网卡配置IP等，所以极少使用.
* 查看none网络模式网络信息
```bah
root@ubuntu:~# docker network inspect none
[
    {
        "Name": "none",
        "Id": "3921a9a0731fe0abd289f583ed4ca780d5003fcba51c298407512c0ce94017ff",
        "Created": "2022-10-22T06:38:25.254740922Z",
        "Scope": "local",
        "Driver": "null",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": []
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
* 创建一个none网络模式的容器
```bash
root@ubuntu:~# docker run -it -d --name busybox1-none-test-container -p 80:80 --net=none busybox sleep 10000000
root@ubuntu:~# docker exec -it busybox1-none-test-container sh
/ # ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
/ # ping 114.114.114.114
PING 114.114.114.114 (114.114.114.114): 56 data bytes
ping: sendto: Network is unreachable
```
### 4.1.4 container模式
* Container模式即容器模式，使用参数 --net=container:目标容器名称/ID 指定,使用此模式创建的容器需指定和一个已经存在的容器共享一个网 络namespace，而不会创建独立的namespace，即新创建的容器不会创建自己的网卡也不会配置自己的IP，而是和一个已经存在的被指定的 目标容器共享对方的IP和端口范围，因此这个容器的端口不能和被指定的目标容器端口冲突，除了网络之外的文件系统、用户信息、进程信息 等仍然保持相互隔离，两个容器的进程可以通过lo网卡及容器IP进行通信。
* 创建container网络模式容器
```bash
root@ubuntu:~# docker run -it -d --name nginx-container -p 80:80 --net=bridge nginx:1.23.2-alpine
root@ubuntu:~# docker run -it -d --name php-container --net=container:nginx-container php:7.4.32-fpm-alpine
# 查看nginx-container 网络信息
root@ubuntu:~# docker exec -it nginx-container sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:14 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1156 (1.1 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
# 查看php-container，发现与nginx-container网络信息一样
root@ubuntu:~# docker exec -it php-container sh
/var/www/html # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:14 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1156 (1.1 KiB)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
root@ubuntu:~# ls /var/run/netns/
root@ubuntu:~# ln -s /var/run/docker/netns/* /var/run/netns/
root@ubuntu:~# ip netns list
f5e7f54509a5 (id: 0)
default
# 通过ip netns 查看容器的网络信息
root@ubuntu:~# ip netns exec f5e7f54509a5 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
25: eth0@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netns default
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
