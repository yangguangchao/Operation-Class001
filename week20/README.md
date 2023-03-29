# 1.Prometheus 基于consul实现服务发现，并总结服务发现过程
## 1.1 consul服务发现过程
* 1. 服务通过consul api接口注册到consul服务中
* 2. Prometheus通过与Consul的交互可以获取到相应target实例的访问信息
* 3. prometheus通过获取的target信息进行数据抓取
## 1.2 prometheus基于consul实现服务发现
```bash
## 基于docker compose部署consul
root@docker1:~# cd docker-compose-deploy-single-consul/
## 下载镜像
root@docker1:~/docker-compose-deploy-single-consul# docker compose pull
## 启动consul服务
root@docker1:~/docker-compose-deploy-single-consul# docker compose up -d
## 查看服务状态
oot@docker1:~/docker-compose-deploy-single-consul# docker compose ps
NAME                IMAGE               COMMAND                  SERVICE             CREATED             STATUS              PORTS
consul-server       consul:latest       "docker-entrypoint.s…"   consul1             40 seconds ago      Up 39 seconds       8300-8302/tcp, 8301-8302/udp, 8600/tcp, 8600/udp, 0.0.0.0:8500->8500/tcp

```
* 访问consul界面
![](pictures/consul-access-01.png)
```bash
## 配置prometheus基于consul实现监控目标发现
root@haproxy1:~# cd /apps/prometheus
root@haproxy1:/apps/prometheus# vi prometheus.yml
  - job_name: consul
    honor_labels: true
    metrics_path: /metrics
    scheme: http
    consul_sd_configs: 
    - server: 172.31.7.120:8500
      services: [] #发现的目标服务名称，空为所有服务，可以写servicea,servcieb,servicec
    relabel_configs: 
    - source_labels: ['__meta_consul_tags']
      target_label: 'product' 
    - source_labels: ['__meta_consul_dc']
      target_label: 'idc' 
    - source_labels: ['__meta_consul_service']
      regex: "consul"
      action: drop
## 检查配置文件
root@haproxy1:/apps/prometheus# ./promtool check config prometheus.yml 
Checking prometheus.yml
 SUCCESS: prometheus.yml is valid prometheus config file syntax
## 重新加载prometheus配置
root@haproxy1:/apps/prometheus# curl -X POST 127.0.0.1:9090/-/reload
## 向consul中注册服务
root@haproxy1:~# curl -X PUT -d '{"id": "node-exporter111","name": "node-exporter111","address": "172.31.7.111","port":39100,"tags": ["node-exporter"],"checks": [{"http": "http://172.31.7.111:39100/","interval": "5s"}]}' http://172.31.7.120:8500/v1/agent/service/register
root@haproxy1:~# curl -X PUT -d '{"id": "node-exporter112","name": "node-exporter112","address": "172.31.7.112","port":39100,"tags": ["node-exporter"],"checks": [{"http": "http://172.31.7.112:39100/","interval": "5s"}]}' http://172.31.7.120:8500/v1/agent/service/register
root@haproxy1:~# curl -X PUT -d '{"id": "node-exporter113","name": "node-exporter113","address": "172.31.7.113","port":39100,"tags": ["node-exporter"],"checks": [{"http": "http://172.31.7.113:39100/","interval": "5s"}]}' http://172.31.7.120:8500/v1/agent/service/register
```
* consul查看服务信息
![](pictures/consul-access-02.png)
* prometheus查看target
![](pictures/prometheus-access-consul-01.png)
* grafana导入11074查看dashboard
![](pictures/grafana-access-01.png)

# 2.Prometheus监控JAVA服务(Tomcat)、Redis、MySQL、HAProxy
## 2.1 prometheus监控java服务(tomcat)
```bash
## 进入到本地资目录
root@k8s-master1:~# cd /usr/local/src/
## 解压prometheus监控压缩包
root@k8s-master1:/usr/local/src# unzip 1.prometheus-case-files.zip
## 进入到tomcat镜像构建目录
root@k8s-master1:/usr/local/src# cd 1.prometheus-case-files/app-monitor-case/tomcat/
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat# cd tomcat-image/
## 更换镜像地址
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/tomcat-image# sed -e 's/harbor.linuxarchitect.io/harbor.yanggc.cn/g' -i build-command.sh
## 编译镜像
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/tomcat-image# bash build-command.sh
## 测试镜像
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/tomcat-image# docker run -it --rm -p 9080:8080 harbor.yanggc.cn/magedu/tomcat-app1:v1
```
* tomcat访问测试
![](pictures/tomcat-access-01.png)
```bash
## 更换tomcat kubernetes资源编排yaml镜像
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/tomcat-image# cd ../yaml/
## 部署tomcat和service
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/yaml# kubectl apply -f ./
deployment.apps/tomcat-deployment created
service/tomcat-service created
## 查看部署的资源
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/yaml# kubectl get pod,svc
NAME                                     READY   STATUS    RESTARTS      AGE
pod/php-apache-7d774bb9d9-7jgz2          1/1     Running   2 (21m ago)   5d
pod/tomcat-deployment-7c7d578ccc-jqk7m   1/1     Running   0             21s

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/kubernetes       ClusterIP   10.100.0.1       <none>        443/TCP        24d
service/php-apache       ClusterIP   10.100.24.49     <none>        80/TCP         5d
service/tomcat-service   NodePort    10.100.253.138   <none>        80:31080/TCP   21s
```
* 浏览器访问tomcat
![](pictures/tomcat-access-02.png)
```bash
## 配置prometheus采集tomcat数据
root@haproxy1:~# vi /apps/prometheus/prometheus.yml 
  - job_name: "tomcat-monitor-metrics"
    static_configs:
    - targets: ["172.31.7.101:31080"]
## 重新加载prometheus配置  
root@haproxy1:~# curl -X POST 127.0.0.1:9090/-/reload
                                  
```
* prometheus查看target
![](pictures/prometheus-tomcat-target-access-01.png)
* grafana导入[dashboard](https://raw.githubusercontent.com/nlighten/tomcat_exporter/master/dashboard/example.json)
![](pictures/prometheus-tomcat-target-access-01.png)
## 2.2 prometheus 监控redis
```bash
## 部署redis服务
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/tomcat/yaml# cd ../../redis/
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/redis# kubectl apply -f yaml/
deployment.apps/redis created
service/redis-exporter-service created
service/redis-redis-service created
## 查看部署的资源
root@k8s-master1:/usr/local/src/1.prometheus-case-files/app-monitor-case/redis# kubectl get pod,svc -n magedu 
NAME                                            READY   STATUS    RESTARTS      AGE
pod/magedu-jenkins-deployment-db96bdb96-8vjpm   1/1     Running   5 (38m ago)   17d
pod/magedu-nginx-deployment-748c55bb6b-m95qh    2/2     Running   6 (39m ago)   6d
pod/redis-79f976c795-jn6w4                      2/2     Running   0             43s

NAME                             TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/magedu-jenkins-service   NodePort   10.100.207.62    <none>        80:38080/TCP                 17d
service/magedu-nginx-service     NodePort   10.100.136.118   <none>        80:30092/TCP,443:30093/TCP   6d
service/redis-exporter-service   NodePort   10.100.64.183    <none>        9121:31082/TCP               43s
service/redis-redis-service      NodePort   10.100.24.5      <none>        6379:31081/TCP               43s
## 配置prometheus采集redis数据
root@haproxy1:~# vi /apps/prometheus/prometheus.yml 
  - job_name: "redis-monitor-metrics"
    static_configs:
    - targets: ["172.31.7.101:31082"]
## prometheus重新加载配置
root@haproxy1:~# curl -X POST 127.0.0.1:9090/-/reload
```
* 查看prometheus target
![](pictures/prometheus-redis-target-01.png)
* grafana导入14615模板验证
![](pictures/grafana-redis-dashboard-01.png)
* grafana导入自定义模板验证
![](pictures/grafana-redis-dashboard-02.png)
## 2.3 prometheus监控mysql
```bash
## 编写docker compose 部署mysql和mysql exporter yaml文件
root@docker1:~# mkdir mysql
root@docker1:~# cd mysql/
root@docker1:~/mysql# vi docker-compose.yml
version: '3'
services:
  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=database
  mysqlexporter:
    image: prom/mysqld-exporter
    ports:
      - "9104:9104"
    environment:
      - DATA_SOURCE_NAME=root:password@(mysql:3306)/database
```
* 验证mysql exporter metrics
![](pictures/mysql-exporter-metrics-access-01.png)
```bash
## 配置prometheus采集mysql数据
root@haproxy1:~# vi /apps/prometheus/prometheus.yml
  - job_name: mysql-monitor-172.31.7.120
    static_configs:
    - targets: ['172.31.7.120:9104']
## 重新加载prometheus配置
root@haproxy1:~# curl -X POST 127.0.0.1:9090/-/reload
```
* prometheus target验证
![](pictures/prometheus-target-mysql-01.png)
* grafana 导入13106模板验证
![](pictures/grafana-mysql-dashboard-01.png)
# 3.总结prometheus基于exporter进行指标数据采集的流程
## 3.1 prometheus采集exporter数据采集流程
![](pictures/exporter-01.png)
* Exporter 是一个采集监控数据并通过 Prometheus 监控规范对外提供数据的组件，它负责从目标系统（Your 服务）搜集数据，并将其转化为 Prometheus 支持的格式
* Prometheus 会周期性地调用 Exporter 提供的 metrics 数据接口来获取数据
## 3.2 Exporter分类
* 直接采集：这一类Exporter直接内置了对Prometheus监控的支持，比如cAdvisor，Kubernetes，Etcd，Gokit等，都直接内置了用于向Prometheus暴露监控数据的端点
* 间接采集：间接采集，原有监控目标并不直接支持Prometheus，因此我们需要通过Prometheus提供的Client Library编写该监控目标的监控采集程序。例如： Mysql Exporter，JMX Exporter，Consul Exporter等。
# 4.Prometheus集合AlertManager实现邮件、钉钉、微信告警
## 4.1 安装alertmanager
## 4.2 邮件告警
## 4.3 钉钉告警
## 4.4 微信告警
## 4.5 告警路由
# 5.基于钉钉告警模板与企业微信告警模板实现自定义告警内容

# 扩展：
## 1.prometheus监控Nginx及Ingress Controller