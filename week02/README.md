# 1. 基于dockerfile，实现分层构建的nginx业务镜像 
## 1.1 制作ubuntu系统基础镜像
选用ubuntu-22.10作为基础镜像，安装需要使用的基本软件
```
root@ubuntu:~# cd /opt/
root@ubuntu:/opt# mkdir -p nginx-img/system-base
root@ubuntu:/opt# cd nginx-img/system-base/
root@ubuntu:/opt/nginx-img/system-base# docker build -t harbor.yanggc.cn/system/ubuntu-base:22.10 .
root@ubuntu:/opt/nginx-img/nginx-base# docker image ls
REPOSITORY                            TAG       IMAGE ID       CREATED          SIZE
harbor.yanggc.cn/system/ubuntu-base   22.10     599eea9c5966   16 minutes ago   445MB
ubuntu                                22.10     0f175c10c2b4   2 days ago       70.2MB
```
制作镜像的Dockerfile:
[Dockerfile](dockerfiles/system-base)

## 1.2 制作nginx基础镜像
```bash
root@ubuntu:/opt/nginx-img/system-base# cd ..
root@ubuntu:/opt/nginx-img# mkdir nginx-base
root@ubuntu:/opt/nginx-img# cd nginx-base/
root@ubuntu:/opt/nginx-img/nginx-base# vi Dockerfile
root@ubuntu:/opt/nginx-img/nginx-base# wget https://nginx.org/download/nginx-1.22.1.tar.gz
root@ubuntu:/opt/nginx-img/nginx-base# docker build -t harbor.yanggc.cn/server/nginx-base:v1.22.1 .
root@ubuntu:/opt/nginx-img/nginx-base# docker image ls
REPOSITORY                            TAG       IMAGE ID       CREATED          SIZE
harbor.yanggc.cn/server/nginx-base    v1.22.1   3db80d4cd23f   2 minutes ago    469MB
harbor.yanggc.cn/system/ubuntu-base   22.10     599eea9c5966   16 minutes ago   445MB
ubuntu                                22.10     0f175c10c2b4   2 days ago       70.2MB
```
制作镜像的Dockerfile:
[Dockerfile](dockerfiles/nginx-base)

## 1.3 制作web服务镜像
```bash
root@ubuntu:/opt/nginx-img/nginx-base# cd ..
root@ubuntu:/opt/nginx-img# mkdir nginx-web1
root@ubuntu:/opt/nginx-img# cd nginx-web1/
root@ubuntu:/opt/nginx-img/nginx-web1# vi Dockerfile
root@ubuntu:/opt/nginx-img/nginx-web1# docker build -t harbor.yanggc.cn/web/my-web:v1.0.0 .
root@ubuntu:/opt/nginx-img/nginx-web1# docker image ls
REPOSITORY                            TAG       IMAGE ID       CREATED          SIZE
harbor.yanggc.cn/web/my-web           v1.0.0    9fafe271ad32   42 seconds ago   474MB
harbor.yanggc.cn/server/nginx-base    v1.22.1   3db80d4cd23f   22 minutes ago   469MB
harbor.yanggc.cn/system/ubuntu-base   22.10     599eea9c5966   36 minutes ago   445MB
ubuntu                                22.10     0f175c10c2b4   2 days ago       70.2MB
```
制作镜像的Dockerfile:
[Dockerfile](dockerfiles/my-web)

## 1.4 启动一个web服务
```bash
root@ubuntu:/opt/nginx-img/nginx-web1# docker run -d --name my-web1 -p 80:80 harbor.yanggc.cn/web/my-web:v1.0.0
c8518ef14a734b0adfa18fd8e19065007a480a08d9428b737eac201382ebfb04
root@ubuntu:/opt/nginx-img/nginx-web1# curl localhost
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>风飘吾独思欢迎您的访问</title>
</head>
<body>
<h2>风飘吾独思欢迎您的访问</h2>
<img src="./images/logo.png">
<p>
    <a href="http://ygc.wiki" target="_blank">app跳转</a>
</p>
</body>
</html>
```
# 2. 基于 docker 实现对容器的 CPU 和内存的资源限制
* 对于Linux 主机，如果没有足够的内容来执行其他重要的系统任务，将会抛出 OOM (Out of Memory Exception,内存溢出、内存泄漏 内存异常), 随后系统会开始杀死进程以释放内存，凡是运行在宿主机的进程都有可能被 kill，包括Dockerd和其它的应用程序，如果重要 的系统进程被Kill,会导致和该进程相关的服务全部宕机。
* 默认情况下，容器没有资源限制，可以使用主机内核调度程序允许的尽可能多的给定资源，Docker提供了控制容器可以限制容器使用多少内存或CPU的 方法，运行docker run命令创建容器的时候可以进行资源限制。
* Docker早期使用cgroupfs进行容器的资源限制管理，然后再调用内核的cgroup进行资源限制，而kubernetes后来使用systemd直接调用cgroup对进程 实现资源限制，等于绕过了docker的cgroupfs的管理，对资源限制更严格、性能更好，因此在kubernetes环境推荐使用systemd进行资源限制。
  * "exec-opts": ["native.cgroupdriver=systemd"],
  * "exec-opts": ["native.cgroupdriver=cgroupfs"],
* 其中许多功能都要求宿主机的内核支持Linux功能，要检查支持，可以使用docker info命令，如果内核中禁用了某项功能，可能会在输出结尾处看到警 告，如下所示:
  * WARNING: No swap limit support
* 解决办法:
  * root@docker-server1:~# vim /etc/default/grub
    * GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0 cgroup_enable=memory swapaccount=1"
  * root@docker-server1:~# sudo update-grub
  * root@docker-server1:~# reboot

## 2.1 限制容器内存
* Docker 可以强制执行硬性内存限制，即只允许容器使用给定的内存大小。
* Docker 也可以执行非硬性内存限制，即容器可以使用尽可能多的内存，除非内核检测到主机上的内存不够用了。
* 大部分的选项取正整数，跟着一个后缀b，k， m，g，，表示字节，千字节，兆字节或千兆字节。
  * Most of these options take a positive integer, followed by a suffix of b, k, m, g, to indicate bytes, kilobytes, megabytes, or gigabytes.
* --oom-score-adj #宿主机kernel对进程使用的内存进行评分，评分最高的将被宿主机内核kill掉(越低越不容易被kill)，可以指定一个 容器的评分为较低的负数，但是不推荐手动指定。
* --oom-kill-disable #对某个容器关闭oom机制。
* 物理内存限制参数:
  * -m or --memory #限制容器可以使用的最大内存量，如果设置此选项，最小存值为4m(4兆字节)
  * --memory-swap #容器可以使用的交换分区大小，必须要在设置了物理内存限制的前提才能设置交换分区的限制
  * --memory-swappiness #设置容器使用交换分区的倾向性，值越高表示越倾向于使用swap分区，范围为0-100，0为能不用就不用，100为 能用就用
  * --kernel-memory #容器可以使用的最大内核内存量，最小为4m，由于内核内存与用户空间内存隔离，因此无法与用户空间内存直接交换， 因此内核内存不足的容器可能会阻塞宿主主机资源，这会对主机和其他容器或者其他服务进程产生影响，因此不要设置内核内存大小。
  * --memory-reservation #允许指定小于--memory的软限制，当Docker检测到主机上的争用或内存不足时会激活该限制，如果使用-- memory-reservation，则必须将其设置为低于--memory才能使其优先。 因为它是软限制，所以不能保证容器不超过限制。
  * --oom-kill-disable #默认情况下，发生OOM时，kernel会杀死容器内进程，但是可以使用--oom-kill-disable参数，可以禁止oom发生在指 定的容器上，即 仅在已设置-m / - memory选项的容器上禁用OOM，如果-m 参数未配置，产生OOM时，主机为了释放内存还会杀死系统进程。
### 2.1.1 物理内存限制验证
假如一个容器未做内存使用限制，则该容器可以利用到系统内存最大空间，默认创建的容器没有做内存资源限制。
```bash
root@ubuntu:~# docker pull lorel/docker-stress-ng
docker run -it --rm lorel/docker-stress-ng --help
```
### 2.1.2 不限制容器内存：
* 启动两个内存工作进程，每个内存工作进程最大允许使用内存256M，且宿主机不限制当前容器最大内存:
```bash
root@ubuntu:~# docker run -it --rm --name magedu-c1 lorel/docker-stress-ng --vm 2 --vm-bytes 256M
root@ubuntu:~# docker stats
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   199.33%   515.8MiB / 3.832GiB   13.14%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   199.33%   515.8MiB / 3.832GiB   13.14%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   199.31%   515.8MiB / 3.832GiB   13.14%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   199.31%   515.8MiB / 3.832GiB   13.14%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   200.17%   515.8MiB / 3.832GiB   13.14%    946B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O     BLOCK I/O   PIDS
c9932bfc4b8e   magedu-c1   200.17%   515.8MiB / 3.832GiB   13.14%    946B / 0B   0B / 0B     5
```
通过观察可以看到容器内存维持在515MiB左右，和容器使用的内存512MiB比较接近

### 2.1.3 限制容器最大内存
```bash
root@ubuntu:~# docker run -it --rm -m 256m --name magedu-c1 lorel/docker-stress-ng --vm 2 --vm-bytes 256M
root@ubuntu:~# docker stats
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   198.40%   90.02MiB / 256MiB   35.17%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   198.40%   90.02MiB / 256MiB   35.17%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   193.54%   228.5MiB / 256MiB   89.27%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   193.54%   228.5MiB / 256MiB   89.27%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   194.93%   163.1MiB / 256MiB   63.72%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   194.93%   163.1MiB / 256MiB   63.72%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   195.14%   127MiB / 256MiB     49.60%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   195.14%   127MiB / 256MiB     49.60%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   195.55%   254.7MiB / 256MiB   99.48%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   195.55%   254.7MiB / 256MiB   99.48%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   191.41%   174.2MiB / 256MiB   68.03%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   191.41%   174.2MiB / 256MiB   68.03%    876B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   191.75%   120.1MiB / 256MiB   46.90%    946B / 0B   0B / 0B     5
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
064e7447c3cc   magedu-c1   191.75%   120.1MiB / 256MiB   46.90%    946B / 0B   0B / 0B     5
```
通过观察可以看到容器内存在256MiB内浮动  

### 2.1.4 限制容器交换分区内存
* 交换分区限制参数 
  * --memory-swap #只有在设置了 --memory 后才会有意义。使用Swap,可以让容器将超出限制部分的内存置换到磁盘上，WARNING：经常将内存交换到磁盘的应用程序会降低性能。 
  * 不同的--memory-swap设置会产生不同的效果： 
    * --memory-swap #值为正数， 那么--memory和--memory-swap都必须要设置，--memory-swap表示你能使用的内存和 swap分区大小的总和，例如： --memory=300m, --memory-swap=1g, 那么该容器能够使用 300m 内存和 700m swap，即--memory是实际物理内存大小值不变，而swap的实际大小计算方式为(--memory-swap)-(--memory)=容器可用swap。 
    * --memory-swap #如果设置为0，则忽略该设置，并将该值视为未设置，即未设置交换分区。
    * --memory-swap #如果等于--memory的值，并且--memory设置为正整数，容器无权访问swap即也有设置交换分区。 
    * --memory-swap #如果设置为unset，如果宿主机开启了swap，则实际容器的swap值为2x(--memory)，即两倍于物理内存 大小，但是并不准确(在容器中使用free命令所看到的swap空间并不精确，毕竟每个容器都可以看到具体大小，但是宿主机的swap是有上限而且不是所有容器看到的累计大小)。 
    * --memory-swap #如果设置为-1，如果宿主机开启了swap，则容器可以使用主机上swap的最大空间
  
* 交换分区限制验证
* 交换分区限制
```bash
root@docker-server1:~# docker run -it --rm -m 256m --memory-swap 512m --name magedu-c1 alpine:3.16.2 sh
root@docker-server1:~# docker stats
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
da0908874f15   magedu-c1   0.00%     872KiB / 256MiB     0.33%     1.18kB / 0B   0B / 0B     1

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
da0908874f15   magedu-c1   0.00%     872KiB / 256MiB     0.33%     1.18kB / 0B   0B / 0B     1
q
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
da0908874f15   magedu-c1   0.00%     872KiB / 256MiB     0.33%     1.18kB / 0B   0B / 0B     1

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
da0908874f15   magedu-c1   0.00%     872KiB / 256MiB     0.33%     1.18kB / 0B   0B / 0B     1

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
da0908874f15   magedu-c1   0.00%     872KiB / 256MiB     0.33%     1.18kB / 0B   0B / 0B     1
```
* 内存大小软限制
```bash
root@docker-server1:~# docker run -it --rm -m 256m --memory-reservation 128m --name magedu-c1 lorel/docker-stress-ng --vm 2 --vm-bytes 256M
root@docker-server1:~# docker stats

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   0.42%     130.2MiB / 256MiB   50.84%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   0.42%     130.2MiB / 256MiB   50.84%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   194.81%   240.3MiB / 256MiB   93.87%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   194.81%   240.3MiB / 256MiB   93.87%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   196.41%   193.2MiB / 256MiB   75.48%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   196.41%   193.2MiB / 256MiB   75.48%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   197.89%   135.6MiB / 256MiB   52.99%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   197.89%   135.6MiB / 256MiB   52.99%    806B / 0B   0B / 0B     5

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT   MEM %     NET I/O     BLOCK I/O   PIDS
6de55bab318a   magedu-c1   197.46%   256MiB / 256MiB     100.00%   876B / 0B   0B / 0B     5
```
通过观察可以看到容器内存在256MiB内浮动 

* 关闭OOM机制
```bash
root@docker-server1:~# docker run -it --rm -m 256m --oom-kill-disable --name magedu-c1 lorel/docker-stress-ng --vm 2 --vm-bytes 256M
WARNING: Your kernel does not support OomKillDisable. OomKillDisable discarded.
stress-ng: info: [1] defaulting to a 86400 second run per stressor
stress-ng: info: [1] dispatching hogs: 2 v
# 提示内核不支持OomKillDisable，OomKillDisable被废弃
```
### 2.1.5 限制容器CPU
* 只给容器分配最多2核宿主机CPU利用率
```bash
root@docker-server1:~# docker run -it --rm --name magedu-c1 --cpus 2 lorel/docker-stress-ng --cpu 4 --vm 4
root@docker-server1:~# top
top - 22:23:42 up 3 min,  2 users,  load average: 4.41, 1.40, 0.50
Tasks: 296 total,   9 running, 287 sleeping,   0 stopped,   0 zombie
%Cpu0  : 49.5 us,  0.3 sy,  0.0 ni, 50.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu1  : 50.3 us,  0.3 sy,  0.0 ni, 49.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  : 50.2 us,  0.3 sy,  0.0 ni, 49.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  : 51.3 us,  0.3 sy,  0.0 ni, 48.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
# 注：CPU资源限制是将分配给容器的2核心分配到了宿主机每一核心CPU上，也就是容器的总CPU值是在宿主机的每一个核心CPU分配了部
分比例。
```
* 将容器运行到指定的CPU上
```bash
#容器服务的cpu绑定到1和3cpu上
root@docker-server1:~# docker run -it --rm --name magedu-c1 --cpus 2 --cpuset-cpus 1,3 lorel/docker-stress-ng --cpu 4 --vm 4
root@docker-server1:~# top
top - 22:27:29 up 6 min,  2 users,  load average: 1.56, 1.19, 0.60
Tasks: 260 total,   9 running, 251 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.6 sy,  0.0 ni, 97.1 id,  0.0 wa,  0.0 hi,  2.3 si,  0.0 st
%Cpu1  : 77.2 us, 22.5 sy,  0.0 ni,  0.3 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu2  :  0.3 us,  0.0 sy,  0.0 ni, 99.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
%Cpu3  : 88.7 us, 11.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
#观察可以看到容器运行只使用了1和3cpu
```
* 基于cpu--shares值(共享值)对CPU进行限制，分别启动两个容器，magedu-c1的--cpu-shares值为1000，magedu-c2的--cpu-shares为500，观察最终效果，--cpu-shares值为1000的magedu-c1的CPU利用率基本是--cpu-shares为500的magedu-c2的两倍：
```bash
root@docker-server1:~# docker run -it --rm --name magedu-c1 --cpu-shares 1000 lorel/docker-stress-ng --cpu 4 --vm 4
docker run -it --rm --name magedu-c2 --cpu-shares 500 lorel/docker-stress-ng --cpu 4 --vm 4
root@docker-server1:~# docker stats
CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   139.47%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   98.03%    1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   139.47%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   98.03%    1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   130.01%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   268.72%   1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   130.01%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   268.72%   1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   131.27%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   266.41%   1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   131.27%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   266.41%   1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13

CONTAINER ID   NAME        CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a4ff8f9e4b14   magedu-c2   131.52%   1.014GiB / 3.799GiB   26.68%    936B / 0B     0B / 0B     13
c983094a539f   magedu-c1   264.74%   1.014GiB / 3.799GiB   26.69%    1.16kB / 0B   0B / 0B     13
```