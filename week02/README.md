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
