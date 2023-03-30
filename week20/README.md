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
```bash
## 下载alertmanager
root@haproxy1:~# cd /apps/
root@haproxy1:/apps# wget https://ghproxy.com/github.com/prometheus/alertmanager/releases/download/v0.25.0/alertmanager-0.25.0.linux-amd64.tar.gz
## 解压安装包
root@haproxy1:/apps# tar xf alertmanager-0.25.0.linux-amd64.tar.gz
## 为alertmanager做一个软连方便后续管理
root@haproxy1:/apps# ln -s alertmanager-0.25.0.linux-amd64 alertmanager
## 创建service文件
root@haproxy1:/apps# vim /etc/systemd/system/alertmanager.service
[Unit]
Description=Prometheus alertmanager
After=network.target

[Service]
ExecStart=/apps/alertmanager/alertmanager --config.file="/apps/alertmanager/alertmanager.yml"

[Install]
WantedBy=multi-user.target
## 启动服务并加入开机启动项
root@haproxy1:/apps# systemctl daemon-reload && systemctl restart alertmanager && systemctl enable alertmanager
Created symlink /etc/systemd/system/multi-user.target.wants/alertmanager.service → /etc/systemd/system/alertmanager.service.
## 查看服务状态
root@haproxy1:/apps# systemctl status alertmanager
```
* 浏览器访问alertmanager
![](pictures/alertmanager-access-01.png)
## 4.2 邮件告警
```bash
## 配置邮箱告警配置
root@haproxy1:/apps# cd alertmanager
root@haproxy1:/apps/alertmanager# vi alertmanager.yml
global:
  resolve_timeout: 2m
  smtp_from: "alert@yanggc.cn"
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_auth_username: "alert@yanggc.cn"
  smtp_auth_password: "******"
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 3m
  receiver: 'mail'

receivers:
  - name: 'mail'
    email_configs:
    - to: 'yangguangchao@yanggc.cn'
      send_resolved: true
## 检查配置文件
root@haproxy1:/apps/alertmanager# ./amtool check-config alertmanager.yml 
Checking 'alertmanager.yml'  SUCCESS
Found:
 - global config
 - route
 - 0 inhibit rules
 - 1 receivers
 - 0 templates
## 重新加载alertmanager配置
root@haproxy1:/apps/alertmanager# curl -lvs -X POST "http://localhost:9093/-/reload"
## 配置prometheus alertmanager和rule_files
root@haproxy1:/apps/alertmanager# cd ../prometheus/
root@haproxy1:/apps/prometheus# vi prometheus.yml
global:

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 127.0.0.1:9093

rule_files:
  - rules/*.yml

scrape_configs:
  - job_name: "prometheus"

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: 'kubernetes-apiservers-monitor' 
    kubernetes_sd_configs: 
    - role: endpoints
      api_server: https://172.31.7.101:6443
      tls_config: 
        insecure_skip_verify: true  
      bearer_token_file: /apps/prometheus/k8s.token 
    scheme: https 
    tls_config: 
      insecure_skip_verify: true 
    bearer_token_file: /apps/prometheus/k8s.token 
    relabel_configs: 
    - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name] 
      action: keep 
      regex: default;kubernetes;https 
    - source_labels: [__address__]
      regex: '(.*):6443'
      replacement: '${1}:9100'
      target_label: __address__
      action: replace
    - source_labels: [__scheme__]
      regex: https
      replacement: http
      target_label: __scheme__
      action: replace

  - job_name: 'kubernetes-nodes-monitor' 
    scheme: http 
    tls_config: 
      insecure_skip_verify: true 
    bearer_token_file: /apps/prometheus/k8s.token 
    kubernetes_sd_configs: 
    - role: node 
      api_server: https://172.31.7.101:6443 
      tls_config: 
        insecure_skip_verify: true 
      bearer_token_file: /apps/prometheus/k8s.token 
    relabel_configs: 
      - source_labels: [__address__] 
        regex: '(.*):10250' 
        replacement: '${1}:9100' 
        target_label: __address__ 
        action: replace 
      - source_labels: [__meta_kubernetes_node_label_failure_domain_beta_kubernetes_io_region] 
        regex: '(.*)' 
        replacement: '${1}' 
        action: replace 
        target_label: LOC 
      - source_labels: [__meta_kubernetes_node_label_failure_domain_beta_kubernetes_io_region] 
        regex: '(.*)' 
        replacement: 'NODE' 
        action: replace 
        target_label: Type 
      - source_labels: [__meta_kubernetes_node_label_failure_domain_beta_kubernetes_io_region] 
        regex: '(.*)' 
        replacement: 'K8S-test' 
        action: replace 
        target_label: Env 
      - action: labelmap 
        regex: __meta_kubernetes_node_label_(.+) 

  - job_name: 'kube-cadvisor'
    kubernetes_sd_configs:
    - api_server: https://172.31.7.101:6443 
      role: node
      bearer_token_file: /apps/prometheus/k8s.token
      tls_config:
        insecure_skip_verify: true    
    bearer_token_file: /apps/prometheus/k8s.token
    tls_config:
      insecure_skip_verify: true 
    scheme: https
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - target_label: __address__
      replacement: 172.31.7.101:6443
    - source_labels: [__meta_kubernetes_node_name]
      regex: (.+)
      target_label: __metrics_path__
      replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
## 配置prometheus告警规则
root@haproxy1:/apps/prometheus# mkdir rules
root@haproxy1:/apps/prometheus# cd rules/
root@haproxy1:/apps/prometheus/rules# vi server_rules.yml
groups:
  - name: alertmanager_pod.rules
    rules:
    - alert: Pod_all_cpu_usage
      expr: (sum by(name)(rate(container_cpu_usage_seconds_total{image!=""}[5m]))*100) > 1
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-cpu
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} CPU 资源利用率大于 10% , (current value is {{ $value }})
        summary: Dev CPU 负载告警

    - alert: Pod_all_memory_usage  
      #expr: sort_desc(avg by(name)(irate(container_memory_usage_bytes{name!=""}[5m]))*100) > 10  #内存大于10%
      expr: sort_desc(avg by(name)(irate(container_memory_usage_bytes{name!=""}[5m]))*1000) > 1  #内存大于10%
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-memory
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} Memory 资源利用率大于 2G , (current value is {{ $value }})
        summary: Dev Memory 负载告警

    - alert: Pod_all_network_receive_usage
      expr: sum by (name)(irate(container_network_receive_bytes_total{container_name="POD"}[1m])) > 1
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-network-receive
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} network_receive 资源利用率大于 50M , (current value is {{ $value }})

    - alert: node内存可用大小 
      expr: node_memory_MemFree_bytes > 1 #故意写错的
      for: 10s
      labels:
        severity: critical
        project: node
      annotations:
        description: node节点 {{ $labels.name }} 的可用内存大于1字节，当前值 {{ $value }}

    - alert: 磁盘容量
      expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 5  #磁盘容量利用率大于80%
      for: 2s
      labels:
        severity: critical
      annotations:
        summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
        description: "{{$labels.mountpoint }} 磁盘分区使用大于5%(目前使用:{{$value}}%)"

    - alert: 磁盘容量
      expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 3 #磁盘容量利用率大于60%
      for: 2s
      labels:
        severity: warning
      annotations:
        summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
        description: "{{$labels.mountpoint }} 磁盘分区使用大于3%(目前使用:{{$value}}%)"
## 重新加载prometheus配置
root@haproxy1:/apps/prometheus/rules# curl -X POST 127.0.0.1:9090/-/reload
## 查看当前告警
root@haproxy1:/apps/prometheus/rules# cd /apps/alertmanager/
root@haproxy1:/apps/alertmanager# ./amtool alert --alertmanager.url=http://127.0.0.1:9093
Alertname             Starts At                Summary           State   
磁盘容量                  2023-03-30 14:32:05 UTC  /boot 磁盘分区使用率过高！  active  
磁盘容量                  2023-03-30 14:32:05 UTC  / 磁盘分区使用率过高！      active  
磁盘容量                  2023-03-30 14:32:05 UTC  /boot 磁盘分区使用率过高！  active  
```
* prometheus alert查看
![](pictures/prometheus-alerts-access-01.png)
* alertmanager alert查看
![](pictures/alertmanager-alerting-access-02.png)
* 报警邮件查看
![](pictures/alert-email-access-01.png)
* 常用prometheus告警rules
  * [node roles](rules/node.yml)
  * [mysql roles](rules/mysql.yml)
  * [nginx rules](rules/nginx.yml)

## 4.3 钉钉告警
```bash
## 脚本测试钉钉机器人接口
root@haproxy1:~# mkdir dingtalk
root@haproxy1:~# cd dingtalk/
root@haproxy1:~/dingtalk# vi dingding-keywords.sh
#!/bin/bash
source /etc/profile
MESSAGE=$1
/usr/bin/curl -X "POST" 'https://oapi.dingtalk.com/robot/send?access_token=***' \
-H 'Content-Type: application/json' \
-d '{"msgtype": "text",
     "text": {
         "content": "'${MESSAGE}'"
     }
}'

root@haproxy1:~/dingtalk# bash dingding-keywords.sh "alertname=node,instaince=172.31.7.101"
{"errcode":0,"errmsg":"ok"}
```
* 查看钉钉
![](pictures/dingtalk-scripts-access-01.png)

```bash
## 下载prometheus-webhook-dingtalk安装包
root@haproxy1:~# cd /apps/
root@haproxy1:/apps# wget https://ghproxy.com/github.com/timonwong/prometheus-webhook-dingtalk/releases/download/v2.1.0/prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz
## 解压安装包
root@haproxy1:/apps# tar -xf prometheus-webhook-dingtalk-2.1.0.linux-amd64.tar.gz
## 软连一个目录
root@haproxy1:/apps# ln -s prometheus-webhook-dingtalk-2.1.0.linux-amd64 prometheus-webhook-dingtalk
## 修改配置文件
root@haproxy1:/apps# cd prometheus-webhook-dingtalk
root@haproxy1:/apps/prometheus-webhook-dingtalk# cp config.example.yml config.yml 
root@haproxy1:/apps/prometheus-webhook-dingtalk# vi config.yml
targets:
  alertname:
    url: https://oapi.dingtalk.com/robot/send?access_token=***
## 配置service文件
root@haproxy1:/apps/prometheus-webhook-dingtalk# vim /etc/systemd/system/prometheus-webhook-dingtalk.service
[Unit]
Description=prometheus-webhook-dingtalk
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/apps/prometheus-webhook-dingtalk/prometheus-webhook-dingtalk --config.file=/apps/prometheus-webhook-dingtalk/config.yml --web.enable-ui --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
## 启动服务
root@haproxy1:/apps/prometheus-webhook-dingtalk# systemctl daemon-reload && systemctl start prometheus-webhook-dingtalk
## 检查服务监听端口
root@haproxy1:/apps/prometheus-webhook-dingtalk# ss -tnl | grep 8060
LISTEN 0      4096               *:8060             *:*  
## alertmanager添加钉钉告警通道
root@haproxy1:~# cd /apps/alertmanager
root@haproxy1:/apps/alertmanager# vi alertmanager.yml
global:
  resolve_timeout: 3m
  smtp_from: "alert@yanggc.cn"
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_auth_username: "alert@yanggc.cn"
  smtp_auth_password: "***"
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 3m
  repeat_interval: 5m
  receiver: 'dingding'

receivers:
  - name: 'mail'
    email_configs:
    - to: 'yangguangchao@yanggc.cn'
      send_resolved: true

  - name: 'dingding'
    webhook_configs:
    - url: 'http://localhost:8060/dingtalk/alertname/send'
## 重新加载alertmanager配置
root@haproxy1:/apps/alertmanager# curl -X POST 127.0.0.1:9093/-/reload
```
* alertmanager查看配置
![](pictures/alertmanager-access-02.png)
* 钉钉查看告警信息
![](pictures/dingtalk-access-02.png)
## 4.4 微信告警
```bash
## alertmanager添加微信告警配置
root@haproxy1:/apps/alertmanager# vi alertmanager.yml
global:
  resolve_timeout: 3m
  smtp_from: "alert@yanggc.cn"
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_auth_username: "alert@yanggc.cn"
  smtp_auth_password: "***"
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 3m
  repeat_interval: 5m
  receiver: 'wechat'

receivers:
  - name: 'mail'
    email_configs:
    - to: 'yangguangchao@yanggc.cn'
      send_resolved: true

  - name: 'dingding'
    webhook_configs:
    - url: 'http://localhost:8060/dingtalk/alertname/send'
      send_resolved: true

  - name: 'wechat'
    wechat_configs:
    - corp_id: ww9f176e23d64d5443
      to_user: '@all'
      agent_id: 1000002
      api_secret: ***
      send_resolved: true
## 重新加载alertmanager配置
root@haproxy1:/apps/alertmanager# curl -X POST 127.0.0.1:9093/-/reload
```
* 企业微信查看告警
![](pictures/alertmanager-wechat-01.jpg)
## 4.5 告警路由
* 将service:node相关的报警发送到微信
* 将service:pod先关的告警发送到邮箱
* 将service:storage相关的发送送到钉钉
```bash
## 修改prometheus rule的label配置
root@haproxy1:/apps/alertmanager# cd ../prometheus/rules/
root@haproxy1:/apps/prometheus/rules# vi server_rules.yml
groups:
  - name: alertmanager_pod.rules
    rules:
    - alert: Pod_all_cpu_usage
      expr: (sum by(name)(rate(container_cpu_usage_seconds_total{image!=""}[5m]))*100) > 1
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-cpu
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} CPU 资源利用率大于 10% , (current value is {{ $value }})
        summary: Dev CPU 负载告警

    - alert: Pod_all_memory_usage  
      #expr: sort_desc(avg by(name)(irate(container_memory_usage_bytes{name!=""}[5m]))*100) > 10  #内存大于10%
      expr: sort_desc(avg by(name)(irate(container_memory_usage_bytes{name!=""}[5m]))*1000) > 1  #内存大于10%
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-memory
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} Memory 资源利用率大于 2G , (current value is {{ $value }})
        summary: Dev Memory 负载告警

    - alert: Pod_all_network_receive_usage
      expr: sum by (name)(irate(container_network_receive_bytes_total{container_name="POD"}[1m])) > 1
      for: 10s
      labels:
        severity: critical
        service: pods
        type: pod-network-receive
        project: myserver
      annotations:
        description: 容器 {{ $labels.name }} network_receive 资源利用率大于 50M , (current value is {{ $value }})

    - alert: node内存可用大小 
      expr: node_memory_MemFree_bytes > 1 #故意写错的
      for: 10s
      labels:
        severity: critical
        service: node
        project: node
      annotations:
        description: node节点 {{ $labels.name }} 的可用内存大于1字节，当前值 {{ $value }}

    - alert: 磁盘容量
      expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 5  #磁盘容量利用率大于80%
      for: 2s
      labels:
        severity: critical
        service: storage
      annotations:
        summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
        description: "{{$labels.mountpoint }} 磁盘分区使用大于5%(目前使用:{{$value}}%)"

    - alert: 磁盘容量
      expr: 100-(node_filesystem_free_bytes{fstype=~"ext4|xfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs"}*100) > 3 #磁盘容量利用率大于60%
      for: 2s
      labels:
        severity: warning
        service: storage
      annotations:
        summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
        description: "{{$labels.mountpoint }} 磁盘分区使用大于3%(目前使用:{{$value}}%)"
## 配置alertmanager告警路由
root@haproxy1:/apps/prometheus/rules# cd ../../alertmanager
root@haproxy1:/apps/alertmanager# vi alertmanager.yml 
global:
  resolve_timeout: 3m
  smtp_from: "alert@yanggc.cn"
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_auth_username: "alert@yanggc.cn"
  smtp_auth_password: "***"
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 3m
  repeat_interval: 5m
  receiver: 'mail'
  routes:
    - receiver: 'wechat'
      group_wait: 2s
      match_re:
        service: node
    - receiver: 'dingding'
      group_wait: 2s
      match_re:
        service: storage

receivers:
  - name: 'mail'
    email_configs:
    - to: 'yangguangchao@yanggc.cn'
      send_resolved: true

  - name: 'dingding'
    webhook_configs:
    - url: 'http://localhost:8060/dingtalk/alertname/send'
      send_resolved: true

  - name: 'wechat'
    wechat_configs:
    - corp_id: ww9f176e23d64d5443
      to_user: '@all'
      agent_id: 1000002
      api_secret: ***
      send_resolved: true
## 重新加载prometheus和alertmanager配置
root@haproxy1:/apps/alertmanager# curl -X POST 127.0.0.1:9090/-/reload
root@haproxy1:/apps/alertmanager# curl -X POST 127.0.0.1:9093/-/reload
```
* 邮箱告警推送
![](pictures/alertmanager-mail-03.png)
* 微信告警推送
![](pictures/alertmanager-wechat-03.jpg)
* 钉钉告警推送
![](pictures/alertmanager-dingtalk-03.png)
# 5.基于钉钉告警模板与企业微信告警模板实现自定义告警内容

# 扩展：
## 1.prometheus监控Nginx及Ingress Controller