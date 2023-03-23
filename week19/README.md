# 1. 总结prometheus服务发现实现过程
## 1.1 服务发现原理
* Prometheus是基于Pull模式的监控系统,无法使用的static_configs的方式静态的定义监控目标
* 为了监控动态目标,需要一个中间的代理人(服务注册中心)
* 代理人掌握着当前所有监控目标的访问信息
* Prometheus向这个代理人询问有哪些监控目标控,就能找到监控的动态目标,这种模式被称为服务发现
## 1.2 服务发现实现过程
* prometheus服务发现模块专门负责发现需要监控的目标采集点(target)信息
* 数据采集模块从服务发现模块订阅该信息，获取到target信息后，其中就包含协议(scheme)、主机地址:端口(instance)、请求路径(metrics_path)、请求参数(params)等
* 然后数据采集模块就可以基于这些信息构建出一个完整的Http Request请求，定时通过pull http协议不断的去目标采集点(target)拉取监控样本数据(sample)
* 最后，将采集到监控样本数据交由TSDB模块进行数据存储。
## 1.3 prometheus常用的服务发现类型
* kubernetes_sd_configs: #基于Kubernetes API实现的服务
发现，让prometheus动态发现kubernetes中被监控的目标 
* static_configs: #静态服务发现，基于prometheus配置文件
指定的监控目标 
* dns_sd_configs: #DNS 服务发现监控目标 
* consul_sd_configs: #Consul 服务发现，基于consul服务动
态发现监控目标 
* file_sd_configs: #基于指定的文件实现服务发现，基于指定
的文件发现监控目标
* prometheus支持的服务发现模式[官网链接](ttps://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file)
## 1.4 kubernetes服务发现过程
* kubernetes的服务发现机制是基于dns的,所有的service都注册到api-server最终存储到etcd中,任何组件想要访问其它服务的话都需要从api-server中获取, 因此etcd中存储有所有元数据
* prometheus要发现kubernetes中的服务, 需要通过list-watch机制从api-server中获取即可, 获取之后prometheus会缓存在自己的内存中.
# 2. 总结prometheus标签重写relabel_configs
* promethues的relabeling（重新修改标签）功能很强大，它能够在抓取到目标实例之前把目标实例的元数据标签动态重新修改，动态添加或者覆盖标签
* prometheus从Kubernetes API动态发现目标(targer)之后，在被发现的target实例中，都包含一些原始的Metadata标签信息，常见默认的标签有：
  * __address__：以<host>:<port> 格式显示目标targets的地址
  * __scheme__：采集的目标服务地址的Scheme形式，HTTP或
者HTTPS
  * __metrics_path__：采集的目标服务的访问路径
* prometheus支持对发现的目标进行label修改，在两个阶段可以重新标记
![](pictures/relabels-01.jpg)
  * relabel_configs ： 在对target进行数据采集之前（比如在采集数据之前重新定义标签信息，如目的IP、目的端口等信息），可以使用relabel_configs添加、修改或删除一些标签、也可以只采集特定目标或过滤目标。
  * metric_relabel_configs：在对target进行数据采集之后，即如果是已经抓取到指标数据时，可以使用metric_relabel_configs做最后的重新标记和过滤。
* label类型：
  * source_labels：源标签，没有经过relabel处理之前的标签名字
  * target_label：通过action处理之后的新的标签名字
  * regex：给定的值或正则表达式匹配，匹配源标签的值
  * replacement：通过分组替换后标签（target_label）对应的/()/() $1:$2
* action简介:
  * replace: #替换标签值，根据regex正则匹配到源标签的值，使用replacement来引用表达式匹配的分组。
  * keep: #满足regex正则条件的实例进行采集，把source_labels中没有匹配到regex正则内容的Target实例丢掉，即只采集匹配成功的实例。
  * drop: #满足regex正则条件的实例不采集，把source_labels中匹配到regex正则内容的Target实例丢掉，即只采集没有匹配到的实例。
  * labelmap: #匹配regex所有标签名称,然后复制匹配标签的值进行分组，可以通过replacement分组引用（${1},${2},…）替代
  * labelkeep: 匹配regex所有标签名称,其它不匹配的标签都将从标签集中删除
  * labeldrop: 匹配regex所有标签名称,其它匹配的标签都将从标签集中删除
  * hashmod: 使用hashmod计算source_labels的Hash值并进行对比，基于自定义的模数取模，以实现对目标进行分类、重新赋值等功能 
# 3. kubernetes环境中部署prometheus并基于kubernetes_sd_config实现pod、node、endpoint等监控目标动态发现
## 3.1 部署cadvisor
```bash
## 切换到本地资源目录
root@k8s-master1:~# cd /usr/local/src/
## 解压prometheus手动部署资源文件
root@k8s-master1:/usr/local/src# unzip 1.prometheus-case-files.zip
## 进入到prometheus资源文件目录
root@k8s-master1:/usr/local/src# cd 1.prometheus-case-files/
## 创建namespace
root@k8s-master1:/usr/local/src/prometheus-case-files# kubectl create ns monitoring
namespace/monitoring created
## 部署cadvisor
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case1-daemonset-deploy-cadvisor.yaml
daemonset.apps/cadvisor created
## 查看pod
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl get pod -n monitoring
NAME             READY   STATUS    RESTARTS   AGE
cadvisor-kg4g6   1/1     Running   0          42s
cadvisor-lb85g   1/1     Running   0          42s
cadvisor-mqrqg   1/1     Running   0          42s
cadvisor-pkddx   1/1     Running   0          42s
cadvisor-qv9vh   1/1     Running   0          42s
cadvisor-wg4hr   1/1     Running   0          42s
```
* 访问cadvisor
![](pictures/cadvisor-access-01.png)

## 3.2 部署node_exporter
```bash
## 应用node_exporter资源编排文件
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case2-daemonset-deploy-node-exporter.yaml
daemonset.apps/node-exporter created
service/node-exporter created

## 查看pod
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl get pod -n monitoring -o wide
NAME                  READY   STATUS    RESTARTS   AGE     IP             NODE           NOMINATED NODE   READINESS GATES
cadvisor-kg4g6        1/1     Running   0          15m     172.31.7.103   172.31.7.103   <none>           <none>
cadvisor-lb85g        1/1     Running   0          15m     172.31.7.113   172.31.7.113   <none>           <none>
cadvisor-mqrqg        1/1     Running   0          15m     172.31.7.112   172.31.7.112   <none>           <none>
cadvisor-pkddx        1/1     Running   0          15m     172.31.7.111   172.31.7.111   <none>           <none>
cadvisor-qv9vh        1/1     Running   0          15m     172.31.7.101   172.31.7.101   <none>           <none>
cadvisor-wg4hr        1/1     Running   0          15m     172.31.7.102   172.31.7.102   <none>           <none>
node-exporter-89t9k   1/1     Running   0          2m32s   172.31.7.103   172.31.7.103   <none>           <none>
node-exporter-g4nl2   1/1     Running   0          2m32s   172.31.7.101   172.31.7.101   <none>           <none>
node-exporter-gzdt2   1/1     Running   0          2m32s   172.31.7.113   172.31.7.113   <none>           <none>
node-exporter-m9nrb   1/1     Running   0          2m32s   172.31.7.112   172.31.7.112   <none>           <none>
node-exporter-n5tvv   1/1     Running   0          2m32s   172.31.7.111   172.31.7.111   <none>           <none>
node-exporter-qslss   1/1     Running   0          2m32s   172.31.7.102   172.31.7.102   <none>           <none>
```
* 访问node_exporter
![](pictures/cadvisor-access-01.png)

## 3.3 prometheus server
```bash
## NFS服务器创建prometheus server数据目录
root@haproxy1:~# mkdir -p /data/k8sdata/prometheusdata
## 修改目录权限
root@haproxy1:~# chown 65534.65534 /data/k8sdata/prometheusdata -R
## 创建monitor serviceaccount
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl create serviceaccount monitor -n monitoring
## 为monitor授予cluster-admin权限
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl create clusterrolebinding monitor-clusterrolebinding -n monitoring --clusterrole=cluster-admin --serviceaccount=monitoring:monitor
## 部署prometheus server configmap资源配置文件
root@k8s-master1:/usr/local/src/1.prometheus-case-files# cat case3-1-prometheus-cfg.yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: monitoring 
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-nginx-pods'
      kubernetes_sd_configs:
      - role: pod
        namespaces: #可选指定namepace，如果不指定就是发现所有的namespace中的pod
          names:
          - myserver
          - magedu
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case3-1-prometheus-cfg.yaml
configmap/prometheus-config created
##  开启prometheus热加载更新配置
root@k8s-master1:/usr/local/src/1.prometheus-case-files# cat case3-2-prometheus-deployment.yaml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitoring
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: registry.cn-hangzhou.aliyuncs.com/zhangshijie/prometheus:v2.42.0 
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h
          - --web.enable-lifecycle
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "512Mi"
            cpu: "500m"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus
          name: prometheus-config
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          nfs:
            server: 172.31.7.109
            path: /data/k8sdata/prometheusdata
## 部署prometheus server和service
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case3-2-prometheus-deployment.yaml
deployment.apps/prometheus-server created
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case3-3-prometheus-svc.yaml 
service/prometheus created
## 查看部署的资源
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl get pod,svc -n monitoring
NAME                                     READY   STATUS    RESTARTS   AGE
pod/cadvisor-kg4g6                       1/1     Running   0          44m
pod/cadvisor-lb85g                       1/1     Running   0          44m
pod/cadvisor-mqrqg                       1/1     Running   0          44m
pod/cadvisor-pkddx                       1/1     Running   0          44m
pod/cadvisor-qv9vh                       1/1     Running   0          44m
pod/cadvisor-wg4hr                       1/1     Running   0          44m
pod/node-exporter-89t9k                  1/1     Running   0          31m
pod/node-exporter-g4nl2                  1/1     Running   0          31m
pod/node-exporter-gzdt2                  1/1     Running   0          31m
pod/node-exporter-m9nrb                  1/1     Running   0          31m
pod/node-exporter-n5tvv                  1/1     Running   0          31m
pod/node-exporter-qslss                  1/1     Running   0          31m
pod/prometheus-server-5bcb94b856-vgffd   1/1     Running   0          6m30s

NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/node-exporter   NodePort   10.100.239.254   <none>        9100:39100/TCP   31m
service/prometheus      NodePort   10.100.106.68    <none>        9090:30090/TCP   2m26s
```
* 访问prometheus
![](pictures/prometheus-access-01.png)

## 3.4 部署grafana server
```bash
## NFS服务器创建grafana server数据目录
root@haproxy1:~# mkdir -p /data/k8sdata/grafana
## 修改目录权限
root@haproxy1:~# chown 472.0 /data/k8sdata/grafana/ -R
## 部署grafana server 资源编排文件
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case5-grafana.yaml 
deployment.apps/grafana created
service/grafana created
## 查看创建的资源
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl get pod,svc -n monitoring 
NAME                                     READY   STATUS    RESTARTS   AGE
pod/cadvisor-kg4g6                       1/1     Running   0          61m
pod/cadvisor-lb85g                       1/1     Running   0          61m
pod/cadvisor-mqrqg                       1/1     Running   0          61m
pod/cadvisor-pkddx                       1/1     Running   0          61m
pod/cadvisor-qv9vh                       1/1     Running   0          61m
pod/cadvisor-wg4hr                       1/1     Running   0          61m
pod/grafana-c5d7c9548-4j2td              1/1     Running   0          2m21s
pod/node-exporter-89t9k                  1/1     Running   0          48m
pod/node-exporter-g4nl2                  1/1     Running   0          48m
pod/node-exporter-gzdt2                  1/1     Running   0          48m
pod/node-exporter-m9nrb                  1/1     Running   0          48m
pod/node-exporter-n5tvv                  1/1     Running   0          48m
pod/node-exporter-qslss                  1/1     Running   0          48m
pod/prometheus-server-5bcb94b856-vgffd   1/1     Running   0          23m

NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/grafana         NodePort   10.100.65.163    <none>        3000:33000/TCP   2m21s
service/node-exporter   NodePort   10.100.239.254   <none>        9100:39100/TCP   48m
service/prometheus      NodePort   10.100.106.68    <none>        9090:30090/TCP   19m
```
* 访问grafana server
![](pictures/grafana-access-01.png)

* grafana添加prometheus数据源
![](pictures/grafana-add-prometheus-source-01.png)

* grafana导入node模板11074并验证数据
![](pictures/grafana-add-prometheus-source-01.png)

* grafana导入pod模板893并验证数据
![](pictures/grafana-add-prometheus-source-01.png)

## 3.5 部署kube-state-metrics
```bash
## 部署 kube-state-metrics资源编排文件
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case6-kube-state-metrics-deploy.yaml 
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
service/kube-state-metrics created
## 查看pod
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl get pod -n kube-system | grep kube-state-metrics
kube-state-metrics-7d6bc6767b-mrlxd        1/1     Running   0              33s
```
* 访问kube-state-metrics
![](pictures/kube-stat-metric-access-01.png)

* prometheus查看kube-state-metrics抓取状态
![](pictures/prometheus-kube-stat-metrics-status-01.png)

* 导入kube-state-metrics模板10856并验证数据
![](pictures/kube-stat-metrics-10856-access-01.png)

# 4. kubernetes环境外部署prometheus并基于kubernetes_sd_config实现pod、node、endpoint等监控目标动态发现
## 4.1 安装prometheus
```bash
## 进入到本地资源目录
root@haproxy1:~# cd /usr/local/src/
## 解压安装包
root@haproxy1:/usr/local/src# tar -xvf prometheus-server-2.40.5-onekey-install.tar.gz
## 安装prometheus
root@haproxy1:/usr/local/src# bash prometheus-install.sh
## 查看服务状态
root@haproxy1:/usr/local/src# systemctl status prometheus 
```
* 访问prometheus target页面
![](pictures/prometheus-access-02.png)

## 4.2 安装grafana

```bash
## 进入到grafana安装目录
root@haproxy1:/usr/local/src# cd /apps/
## 解压安装
root@haproxy1:/apps# tar -xf grafana-enterprise-9.4.7.linux-amd64.tar.gz 
## 软链到grafana目录
root@haproxy1:/apps# ln -s grafana-9.4.7 grafana
## 配置service文件
root@haproxy1:/apps# vi /etc/systemd/system/grafana-server.service
[Unit]
Description=Grafana Server
After=network.target

[Service]
ExecStart=/apps/grafana/bin/grafana-server -homepath=/apps/grafana
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
## 重新加载 systemd的配置文件
root@haproxy1:/apps# systemctl daemon-reload
## 启动grafana
root@haproxy1:/apps# systemctl start grafana-server
## 查看grafana状态
root@haproxy1:/apps# systemctl status grafana-server
```
* 访问grafana
![](pictures/grafana-access-02.png)
* 添加prometheus数据源
![](pictures/grafana-add-datasource-prometheus-02.png)

## 4.3 配置基于kubernetess_sd_config监控kubernetes目标
```bash
## 进入到prometheus资源目录
root@k8s-master1:~# cd /usr/local/src/1.prometheus-case-files/
## 创建serviceaccount并授权
root@k8s-master1:/usr/local/src/1.prometheus-case-files# vi case4-prom-rbac.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: prometheus
  namespace: monitoring
  annotations:
    kubernetes.io/service-account.name: "prometheus"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
    - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
#apiVersion: rbac.authorization.k8s.io/v1beta1
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl apply -f case4-prom-rbac.yaml 
serviceaccount/prometheus created
secret/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
## 查看token
root@k8s-master1:/usr/local/src/1.prometheus-case-files# kubectl describe secrets prometheus -n monitoring
Name:         prometheus
Namespace:    monitoring
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: prometheus
              kubernetes.io/service-account.uid: d202b6d0-41f0-47d5-b528-af87b3f972e6

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1302 bytes
namespace:  10 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlFCOTFLcFZacDFWa3pIWHZVdWJ0aGN5WHd0REx5MHhyMzkzOTVYQ3ZsaUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtb25pdG9yaW5nIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InByb21ldGhldXMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicHJvbWV0aGV1cyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImQyMDJiNmQwLTQxZjAtNDdkNS1iNTI4LWFmODdiM2Y5NzJlNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDptb25pdG9yaW5nOnByb21ldGhldXMifQ.s42NthiRe2-vpCV3e92kkhWZlu16T4zrZtkGa5X6e7x4cjKjVqDeYV3eEPXeKmeqYtB5YJDSWI83-7cNAH40vQnz4RVVi5ZBqCfDT_nS9IPFdevzyEGw2D1Q-pAcm-oiCnmFBGcS_u4hYWny0X4M8YxsgPr9CbZ6Gj3b196DBERJwnFnNkUeSI37motUhiMBH7NGoT6R57JrpB7b130L6ALV1CuWutz839-G8ahGwSWUWKCKuMVeKRW9j2kwAP-HCQ5BMrZiSNgSPpg_Y7B7CZxlf6mJyl0VxXXh-fP2EDKIIp05s0dabg7mJPaXEOuemR-1GU8KagIIhFXFxH8VEw

## 配置prometheus
root@haproxy1:~# vi /apps/prometheus/prometheus.yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

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
    #课自定义替换发现的服务器端口、协议等
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

#node节点发现
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

#指定namespace 的pod
  - job_name: 'kubernetes-发现指定namespace的所有pod'
    kubernetes_sd_configs:
    - role: pod
      api_server: https://172.31.7.101:6443
      tls_config:
        insecure_skip_verify: true
      bearer_token_file: /apps/prometheus/k8s.token
      namespaces:
        names:
        - myserver
        - magedu
    relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod_name

#指定Pod发现条件
  - job_name: 'kubernetes-指定发现条件的pod' 
    kubernetes_sd_configs: 
    - role: pod 
      api_server: https://172.31.7.101:6443 
      tls_config: 
        insecure_skip_verify: true 
      bearer_token_file: /apps/prometheus/k8s.token 
    relabel_configs: 
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape] 
      action: keep 
      regex: true 
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path] 
      action: replace 
      target_label: __metrics_path__ 
      regex: (.+) 
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port] 
      action: replace 
      regex: ([^:]+)(?::\d+)?;(\d+) 
      replacement: $1:$2 
      target_label: __address__ 
    - action: labelmap 
      regex: __meta_kubernetes_pod_label_(.+) 
    - source_labels: [__meta_kubernetes_namespace] 
      action: replace 
      target_label: kubernetes_namespace 
    - source_labels: [__meta_kubernetes_pod_name] 
      action: replace 
      target_label: kubernetes_pod_name 
    - source_labels: [__meta_kubernetes_pod_label_pod_template_hash] 
      regex: '(.*)' 
      replacement: 'K8S-test' 
      action: replace 
      target_label: Env
## 创建token文件
root@haproxy1:~# vi /apps/prometheus/k8s.token
eyJhbGciOiJSUzI1NiIsImtpZCI6IlFCOTFLcFZacDFWa3pIWHZVdWJ0aGN5WHd0REx5MHhyMzkzOTVYQ3ZsaUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtb25pdG9yaW5nIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InByb21ldGhldXMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicHJvbWV0aGV1cyIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImQyMDJiNmQwLTQxZjAtNDdkNS1iNTI4LWFmODdiM2Y5NzJlNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDptb25pdG9yaW5nOnByb21ldGhldXMifQ.s42NthiRe2-vpCV3e92kkhWZlu16T4zrZtkGa5X6e7x4cjKjVqDeYV3eEPXeKmeqYtB5YJDSWI83-7cNAH40vQnz4RVVi5ZBqCfDT_nS9IPFdevzyEGw2D1Q-pAcm-oiCnmFBGcS_u4hYWny0X4M8YxsgPr9CbZ6Gj3b196DBERJwnFnNkUeSI37motUhiMBH7NGoT6R57JrpB7b130L6ALV1CuWutz839-G8ahGwSWUWKCKuMVeKRW9j2kwAP-HCQ5BMrZiSNgSPpg_Y7B7CZxlf6mJyl0VxXXh-fP2EDKIIp05s0dabg7mJPaXEOuemR-1GU8KagIIhFXFxH8VEw
## 重新加载prometheus配置文件
root@haproxy1:~# curl -X POST 127.0.0.1:9090/-/reload
```
* prometheus访问测试
![](pictures/prometheus-access-03.png)
* 由于prometheus在集群外部无法访问集群内的pod，所以pod只能发现不能采集信息，后面研究一下使用kubernetes api代理方式监控
# 扩展： 
## 1.基于HPA控制器对pod副本实现弹性伸缩
```bash
## 解压hpa资源压缩包
root@k8s-master1:/usr/local/src# unzip 2.hpa_case-smetrics-server-0.6.1-case.zip
## metrics-server
root@k8s-master1:/usr/local/src# cd 2.hpa-metrics-server-0.6.1-case/
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl apply -f metrics-server-v0.6.1.yaml 
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
## 查看pod
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl get pod -n kube-system 
NAME                                       READY   STATUS    RESTARTS      AGE
calico-kube-controllers-6878c555d5-wmvx6   1/1     Running   6 (78m ago)   19d
calico-node-7c6v5                          1/1     Running   5 (78m ago)   19d
calico-node-fjzdr                          1/1     Running   5 (78m ago)   19d
calico-node-jbmdc                          1/1     Running   6 (77m ago)   19d
calico-node-nwf4x                          1/1     Running   5 (78m ago)   19d
calico-node-vvm42                          1/1     Running   5 (78m ago)   19d
calico-node-xdrld                          1/1     Running   5 (78m ago)   19d
coredns-5f8b57cc79-xf6wh                   1/1     Running   5 (78m ago)   19d
kube-state-metrics-7d6bc6767b-mrlxd        1/1     Running   2 (77m ago)   22h
metrics-server-b4bcb9f7f-2t6xk             1/1     Running   0             35s
## 验证安装
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
172.31.7.101   160m         8%     1669Mi          46%       
172.31.7.102   137m         6%     1582Mi          44%       
172.31.7.103   147m         7%     1507Mi          41%       
172.31.7.111   95m          4%     1142Mi          31%       
172.31.7.112   111m         5%     1449Mi          40%       
172.31.7.113   108m         5%     1944Mi          54%       
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl top pod -A
NAMESPACE              NAME                                         CPU(cores)   MEMORY(bytes)   
kube-system            calico-kube-controllers-6878c555d5-wmvx6     3m           19Mi            
kube-system            calico-node-7c6v5                            30m          130Mi           
kube-system            calico-node-fjzdr                            26m          131Mi           
kube-system            calico-node-jbmdc                            27m          131Mi           
kube-system            calico-node-nwf4x                            28m          132Mi           
kube-system            calico-node-vvm42                            28m          130Mi           
kube-system            calico-node-xdrld                            29m          132Mi           
kube-system            coredns-5f8b57cc79-xf6wh                     3m           14Mi            
kube-system            kube-state-metrics-7d6bc6767b-mrlxd          2m           10Mi            
kube-system            metrics-server-b4bcb9f7f-2t6xk               5m           17Mi            
kubernetes-dashboard   dashboard-metrics-scraper-557bc75fb9-jkklm   1m           3Mi             
kubernetes-dashboard   kubernetes-dashboard-6c67d8fb4c-p25p7        1m           9Mi             
magedu                 magedu-jenkins-deployment-db96bdb96-8vjpm    2m           557Mi           
magedu                 magedu-nginx-deployment-748c55bb6b-m95qh     1m           6Mi             
monitoring             cadvisor-kg4g6                               27m          58Mi            
monitoring             cadvisor-lb85g                               52m          64Mi            
monitoring             cadvisor-mqrqg                               38m          61Mi            
monitoring             cadvisor-pkddx                               40m          66Mi            
monitoring             cadvisor-qv9vh                               27m          59Mi            
monitoring             cadvisor-wg4hr                               29m          57Mi            
monitoring             grafana-c5d7c9548-4j2td                      1m           48Mi            
monitoring             node-exporter-89t9k                          7m           9Mi             
monitoring             node-exporter-g4nl2                          5m           9Mi             
monitoring             node-exporter-gzdt2                          4m           9Mi             
monitoring             node-exporter-m9nrb                          3m           9Mi             
monitoring             node-exporter-n5tvv                          5m           8Mi             
monitoring             node-exporter-qslss                          5m           9Mi             
monitoring             prometheus-server-5bcb94b856-vgffd           10m          255Mi
##hpa 应用yaml文件
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# vi php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.cn-beijing.aliyuncs.com/ygc/hpa-example:v1.0.0
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache

## HPA配置
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# vi php-apache-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1 #api版本
    kind: Deployment #资源类型
    name: php-apache #资源名称
  maxReplicas: 5
  minReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60 #HPA 控制器会维持扩缩目标中的 Pods 的平均资源利用率在 60%
## 部署应用和hpa配置
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl apply -f php-apache.yaml -f php-apache-hpa.yaml 
deployment.apps/php-apache created
service/php-apache created
horizontalpodautoscaler.autoscaling/php-apache created
## 压测php-apache应用，等到应用副本达到最大停止压测
root@k8s-master1:~# kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
## 查看hpa扩缩容变化
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/60%    1         5         1          83s
php-apache   Deployment/php-apache   8%/60%    1         5         1          3m1s
php-apache   Deployment/php-apache   248%/60%   1         5         1          3m16s
php-apache   Deployment/php-apache   246%/60%   1         5         4          3m31s
php-apache   Deployment/php-apache   141%/60%   1         5         5          3m46s
php-apache   Deployment/php-apache   65%/60%    1         5         5          4m1s
php-apache   Deployment/php-apache   49%/60%    1         5         5          4m16s
php-apache   Deployment/php-apache   13%/60%    1         5         5          4m31s
php-apache   Deployment/php-apache   0%/60%     1         5         5          4m46s
php-apache   Deployment/php-apache   0%/60%     1         5         5          9m17s
php-apache   Deployment/php-apache   0%/60%     1         5         2          9m32s
php-apache   Deployment/php-apache   0%/60%     1         5         1          9m47s

## hpa配置添加扩缩容策略
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# vi php-apache-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1 #api版本
    kind: Deployment #资源类型
    name: php-apache #资源名称
  maxReplicas: 5
  minReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60 #HPA 控制器会维持扩缩目标中的 Pods 的平均资源利用率在 60%
  behavior:
    scaleDown: #缩容策略配置
      stabilizationWindowSeconds: 60 #稳定窗口时间60秒
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30 #允许在三十秒内做多缩容一个副本
      - type: Percent
        value: 20
        periodSeconds: 30 #允许在三十秒内最多缩容当前副本个数的百分之二十
      selectPolicy: Min #缩容选择策略，选择缩容最小的副本
    scaleUp: #扩容策略配置
      stabilizationWindowSeconds: 30 #稳定窗口时间30秒
      policies:
      - type: Pods
        value: 2
        periodSeconds: 15 #允许十五秒内最多扩容两个pod
      - type: Percent
        value: 300
        periodSeconds: 15 #允许十五秒内最多扩容当前副本个数的百分之三百
      selectPolicy: Max

root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl apply -f php-apache-hpa.yaml
horizontalpodautoscaler.autoscaling/php-apache configured

## 压测php-apache应用，等到应用副本达到最大停止压测
root@k8s-master1:~# kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
## 由于一个压测任务应用副本达不到最大值，再添加一个压测任务
root@k8s-master1:~# kubectl run -i --tty load-generator2 --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
## 查看添加扩缩容策略应用副本伸缩变化
root@k8s-master1:/usr/local/src/2.hpa-metrics-server-0.6.1-case# kubectl get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/60%    1         5         1          11m
php-apache   Deployment/php-apache   134%/60%   1         5         1          12m
php-apache   Deployment/php-apache   242%/60%   1         5         1          12m
php-apache   Deployment/php-apache   126%/60%   1         5         3          12m
php-apache   Deployment/php-apache   81%/60%    1         5         3          13m
php-apache   Deployment/php-apache   80%/60%    1         5         3          13m
php-apache   Deployment/php-apache   70%/60%    1         5         4          13m
php-apache   Deployment/php-apache   60%/60%    1         5         4          14m
php-apache   Deployment/php-apache   59%/60%    1         5         4          14m
php-apache   Deployment/php-apache   61%/60%    1         5         4          14m
php-apache   Deployment/php-apache   58%/60%    1         5         4          14m
php-apache   Deployment/php-apache   60%/60%    1         5         4          15m
php-apache   Deployment/php-apache   60%/60%    1         5         4          15m
php-apache   Deployment/php-apache   60%/60%    1         5         4          15m
php-apache   Deployment/php-apache   77%/60%    1         5         4          15m
php-apache   Deployment/php-apache   100%/60%   1         5         4          16m
php-apache   Deployment/php-apache   94%/60%    1         5         5          16m
php-apache   Deployment/php-apache   88%/60%    1         5         5          16m
php-apache   Deployment/php-apache   87%/60%    1         5         5          16m
php-apache   Deployment/php-apache   85%/60%    1         5         5          17m
php-apache   Deployment/php-apache   38%/60%    1         5         5          17m
php-apache   Deployment/php-apache   0%/60%     1         5         5          17m
php-apache   Deployment/php-apache   0%/60%     1         5         5          18m
php-apache   Deployment/php-apache   0%/60%     1         5         4          18m
php-apache   Deployment/php-apache   0%/60%     1         5         4          18m
php-apache   Deployment/php-apache   0%/60%     1         5         3          18m
php-apache   Deployment/php-apache   0%/60%     1         5         3          19m
php-apache   Deployment/php-apache   0%/60%     1         5         2          19m
php-apache   Deployment/php-apache   0%/60%     1         5         2          19m
php-apache   Deployment/php-apache   0%/60%     1         5         1          19m
```
* 通过对比测试，发现加入扩缩容策略，HPA扩缩容更加平滑