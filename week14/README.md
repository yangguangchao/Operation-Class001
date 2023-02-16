## 1.2 
### 1.2.1 安装helm
```bash
root@k8s-master1:~# wget https://get.helm.sh/helm-v3.11.1-linux-amd64.tar.gz
root@k8s-master1:~# tar -xzvf helm-v3.11.1-linux-amd64.tar.gz 
linux-amd64/
linux-amd64/helm
linux-amd64/LICENSE
linux-amd64/README.md
root@k8s-master1:~# mv linux-amd64/helm /usr/local/bin/helm
root@k8s-master1:~# helm help
The Kubernetes package manager

Common actions for Helm:

- helm search:    search for charts
- helm pull:      download a chart to your local directory to view
- helm install:   upload the chart to Kubernetes
- helm list:      list releases of charts

Environment variables:

| Name                               | Description                                                                                       |
|------------------------------------|---------------------------------------------------------------------------------------------------|
| $HELM_CACHE_HOME                   | set an alternative location for storing cached files.                                             |
| $HELM_CONFIG_HOME                  | set an alternative location for storing Helm configuration.                                       |
| $HELM_DATA_HOME                    | set an alternative location for storing Helm data.                                                |
| $HELM_DEBUG                        | indicate whether or not Helm is running in Debug mode                                             |
| $HELM_DRIVER                       | set the backend storage driver. Values are: configmap, secret, memory, sql.                       |
| $HELM_DRIVER_SQL_CONNECTION_STRING | set the connection string the SQL storage driver should use.                                      |
| $HELM_MAX_HISTORY                  | set the maximum number of helm release history.                                                   |
| $HELM_NAMESPACE                    | set the namespace used for the helm operations.                                                   |
| $HELM_NO_PLUGINS                   | disable plugins. Set HELM_NO_PLUGINS=1 to disable plugins.                                        |
| $HELM_PLUGINS                      | set the path to the plugins directory                                                             |
| $HELM_REGISTRY_CONFIG              | set the path to the registry config file.                                                         |
| $HELM_REPOSITORY_CACHE             | set the path to the repository cache directory                                                    |
| $HELM_REPOSITORY_CONFIG            | set the path to the repositories file.                                                            |
| $KUBECONFIG                        | set an alternative Kubernetes configuration file (default "~/.kube/config")                       |
| $HELM_KUBEAPISERVER                | set the Kubernetes API Server Endpoint for authentication                                         |
| $HELM_KUBECAFILE                   | set the Kubernetes certificate authority file.                                                    |
| $HELM_KUBEASGROUPS                 | set the Groups to use for impersonation using a comma-separated list.                             |
| $HELM_KUBEASUSER                   | set the Username to impersonate for the operation.                                                |
| $HELM_KUBECONTEXT                  | set the name of the kubeconfig context.                                                           |
| $HELM_KUBETOKEN                    | set the Bearer KubeToken used for authentication.                                                 |
| $HELM_KUBEINSECURE_SKIP_TLS_VERIFY | indicate if the Kubernetes API server's certificate validation should be skipped (insecure)       |
| $HELM_KUBETLS_SERVER_NAME          | set the server name used to validate the Kubernetes API server certificate                        |
| $HELM_BURST_LIMIT                  | set the default burst limit in the case the server contains many CRDs (default 100, -1 to disable)|

Helm stores cache, configuration, and data based on the following configuration order:

- If a HELM_*_HOME environment variable is set, it will be used
- Otherwise, on systems supporting the XDG base directory specification, the XDG variables will be used
- When no other location is set a default location will be used based on the operating system

By default, the default directories depend on the Operating System. The defaults are listed below:

| Operating System | Cache Path                | Configuration Path             | Data Path               |
|------------------|---------------------------|--------------------------------|-------------------------|
| Linux            | $HOME/.cache/helm         | $HOME/.config/helm             | $HOME/.local/share/helm |
| macOS            | $HOME/Library/Caches/helm | $HOME/Library/Preferences/helm | $HOME/Library/helm      |
| Windows          | %TEMP%\helm               | %APPDATA%\helm                 | %APPDATA%\helm          |

Usage:
  helm [command]

Available Commands:
  completion  generate autocompletion scripts for the specified shell
  create      create a new chart with the given name
  dependency  manage a chart's dependencies
  env         helm client environment information
  get         download extended information of a named release
  help        Help about any command
  history     fetch release history
  install     install a chart
  lint        examine a chart for possible issues
  list        list releases
  package     package a chart directory into a chart archive
  plugin      install, list, or uninstall Helm plugins
  pull        download a chart from a repository and (optionally) unpack it in local directory
  push        push a chart to remote
  registry    login to or logout from a registry
  repo        add, list, remove, update, and index chart repositories
  rollback    roll back a release to a previous revision
  search      search for a keyword in charts
  show        show information of a chart
  status      display the status of the named release
  template    locally render templates
  test        run tests for a release
  uninstall   uninstall a release
  upgrade     upgrade a release
  verify      verify that a chart at the given path has been signed and is valid
  version     print the client version information

Flags:
      --burst-limit int                 client-side default throttling limit (default 100)
      --debug                           enable verbose output
  -h, --help                            help for helm
      --kube-apiserver string           the address and the port for the Kubernetes API server
      --kube-as-group stringArray       group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --kube-as-user string             username to impersonate for the operation
      --kube-ca-file string             the certificate authority file for the Kubernetes API server connection
      --kube-context string             name of the kubeconfig context to use
      --kube-insecure-skip-tls-verify   if true, the Kubernetes API server's certificate will not be checked for validity. This will make your HTTPS connections insecure
      --kube-tls-server-name string     server name to use for Kubernetes API server certificate validation. If it is not provided, the hostname used to contact the server is used
      --kube-token string               bearer token used for authentication
      --kubeconfig string               path to the kubeconfig file
  -n, --namespace string                namespace scope for this request
      --registry-config string          path to the registry config file (default "/root/.config/helm/registry/config.json")
      --repository-cache string         path to the file containing cached repository indexes (default "/root/.cache/helm/repository")
      --repository-config string        path to the file containing repository names and URLs (default "/root/.config/helm/repositories.yaml")

Use "helm [command] --help" for more information about a command.
```
### 1.2.2 安装mysql-operator
```bash
## 添加Charts仓库
root@k8s-master1:~# helm repo add mysql-operator https://mysql.github.io/mysql-operator/
"mysql-operator" has been added to your repositories
## 搜索mysql operator版本
root@k8s-master1:~# helm search repo mysql-operator
NAME                              	CHART VERSION	APP VERSION 	DESCRIPTION                                       
mysql-operator/mysql-operator     	2.0.8        	8.0.32-2.0.8	MySQL Operator Helm Chart for deploying MySQL I...
mysql-operator/mysql-innodbcluster	2.0.8        	8.0.32      	MySQL InnoDB Cluster Helm Chart for deploying M...
## 安装Operator
root@k8s-master1:~# helm install mysql-operator mysql-operator/mysql-operator --namespace mysql-operator --create-namespace
NAME: mysql-operator
LAST DEPLOYED: Fri Feb 17 00:12:30 2023
NAMESPACE: mysql-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Create an MySQL InnoDB Cluster by executing:
1. When using a source distribution / git clone: `helm install [cluster-name] -n [ns-name] ~/helm/mysql-innodbcluster`
2. When using the Helm repo from ArtifactHub
2.1 With self signed certificates
    export NAMESPACE="your-namespace"
    # in case the namespace doesn't exist, please pass --create-namespace
    helm install my-mysql-innodbcluster mysql-operator/mysql-innodbcluster -n $NAMESPACE \
        --version 2.0.8 \
        --set credentials.root.password=">-0URS4F3P4SS" \
        --set tls.useSelfSigned=true

2.2 When you have own CA and TLS certificates
        export NAMESPACE="your-namespace"
        export CLUSTER_NAME="my-mysql-innodbcluster"
        export CA_SECRET="$CLUSTER_NAME-ca-secret"
        export TLS_SECRET="$CLUSTER_NAME-tls-secret"
        export ROUTER_TLS_SECRET="$CLUSTER_NAME-router-tls-secret"
        # Path to ca.pem, server-cert.pem, server-key.pem, router-cert.pem and router-key.pem
        export CERT_PATH="/path/to/your/ca_and_tls_certificates"

        kubectl create namespace $NAMESPACE

        kubectl create secret generic $CA_SECRET \
            --namespace=$NAMESPACE --dry-run=client --save-config -o yaml \
            --from-file=ca.pem=$CERT_PATH/ca.pem \
        | kubectl apply -f -

        kubectl create secret tls $TLS_SECRET \
            --namespace=$NAMESPACE --dry-run=client --save-config -o yaml \
            --cert=$CERT_PATH/server-cert.pem --key=$CERT_PATH/server-key.pem \
        | kubectl apply -f -

        kubectl create secret tls $ROUTER_TLS_SECRET \
            --namespace=$NAMESPACE --dry-run=client --save-config -o yaml \
            --cert=$CERT_PATH/router-cert.pem --key=$CERT_PATH/router-key.pem \
        | kubectl apply -f -

        helm install my-mysql-innodbcluster mysql-operator/mysql-innodbcluster -n $NAMESPACE \
        --version 2.0.8 \
        --set credentials.root.password=">-0URS4F3P4SS" \
        --set tls.useSelfSigned=false \
        --set tls.caSecretName=$CA_SECRET \
        --set tls.serverCertAndPKsecretName=$TLS_SECRET \
        --set tls.routerCertAndPKsecretName=$ROUTER_TLS_SECRET
## 创建用户名和密码的secret
root@k8s-master1:~# kubectl create secret generic mysql-cluster01-passwd \
        --from-literal=rootUser=root \
        --from-literal=rootHost=% \
        --from-literal=rootPassword="Ygx@2023"
secret/mysql-cluster01-passwd created
## 定义InnoDBCluster资源
root@k8s-master1:~# vi mysql-cluster01-default.yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: mysql-cluster01
spec:
  # 使用创建的secret
  secretName: mysql-cluster01-passwd
  tlsUseSelfSigned: true
  # MySQL实例个数
  instances: 3
  # MySQL Router配置
  router:
    instances: 1
  # 数据持久化配置
  datadirVolumeClaimTemplate:
    accessModes: 
      - ReadWriteOnce
    resources:
      requests:
        # 定义存储大小
        storage: 40Gi
    # 定义存储类
    storageClassName: nfs-csi
## 创建InnoDBCluster资源
## 使用kubectl创建资源
root@k8s-master1:~# kubectl apply -f mysql-cluster01-default.yaml 
innodbcluster.mysql.oracle.com/mysql-cluster01 created
## 实时查看资源的状态
root@k8s-master1:~# kubectl get innodbcluster --watch
## 过一段时间后，可以看到资源已经ONLINE状态:
root@k8s-master1:~# kubectl get innodbcluster --watch
NAME              STATUS    ONLINE   INSTANCES   ROUTERS   AGE
mysql-cluster01   PENDING   0        3           1         16s
mysql-cluster01   PENDING   0        3           1         2m33s
mysql-cluster01   INITIALIZING   0        3           1         2m33s
mysql-cluster01   INITIALIZING   0        3           1         2m33s
mysql-cluster01   INITIALIZING   0        3           1         2m33s
mysql-cluster01   INITIALIZING   0        3           1         2m33s
mysql-cluster01   INITIALIZING   0        3           1         3m41s
mysql-cluster01   ONLINE_PARTIAL   1        3           1         3m41s
mysql-cluster01   ONLINE_PARTIAL   1        3           1         4m3s
mysql-cluster01   ONLINE_PARTIAL   2        3           1         4m7s
mysql-cluster01   ONLINE           2        3           1         7m5s
mysql-cluster01   ONLINE           3        3           1         7m9s
```
### 1.2.3 使用mysql shell进行连接测试
```bash
root@k8s-master1:~# kubectl run --rm -it myshell \
--image=mysql/mysql-operator \
-- mysqlsh root@mysql-cluster01-instances \
--sql \
--password=Ygx@2023
If you don't see a command prompt, try pressing enter.

 MySQL  mysql-cluster01-instances:33060+ ssl  SQL > show databases;
+-------------------------------+
| Database                      |
+-------------------------------+
| information_schema            |
| mysql                         |
| mysql_innodb_cluster_metadata |
| performance_schema            |
| sys                           |
+-------------------------------+
5 rows in set (0.0016 sec)
```
### 1.2.4 创建wordpress需要用的数据库和账号
```bash
root@k8s-master1:~# kubectl run --rm -it myshell --image=mysql/mysql-operator -- mysqlsh root@mysql-cluster01 --sql --password=Ygx@2023
If you don't see a command prompt, try pressing enter.

 MySQL  mysql-cluster01:33060+ ssl  SQL > create database wordpress;
Query OK, 1 row affected (0.0085 sec)
 MySQL  mysql-cluster01:33060+ ssl  SQL >create user 'wordpress'@'%' identified with mysql_native_password BY 'Wordpress@2023';
Query OK, 0 rows affected (0.0082 sec)
 MySQL  mysql-cluster01:33060+ ssl  SQL > grant all privileges on *.* to 'wordpress'@'%' with grant option;
Query OK, 0 rows affected (0.0053 sec)
 MySQL  mysql-cluster01:33060+ ssl  SQL >  flush privileges;
```
### 1.2.5 部署wordpress
```bash
## 进入到mysql资源编排目录
root@k8s-master1:~# cd learning-k8s/wordpress/mysql
## 修改mysql secret为设置的账号和密码
root@k8s-master1:~/learning-k8s/wordpress/mysql# vi 01-secret-mysql.yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: mysql-user-pass
data:
  database.name: d29yZHByZXNz
  root.password: WWdjQDIwMjM=
  user.name: d29yZHByZXNz
  user.password: V29yZHByZXNzQDIwMjM=
## 创建secret
root@k8s-master1:~/learning-k8s/wordpress/mysql# k apply -f 01-secret-mysql.yaml 
secret/mysql-user-pass created
## 查看secret
root@k8s-master1:~/learning-k8s/wordpress/mysql# k get secrets 
NAME                          TYPE     DATA   AGE
mysql-cluster01-backup        Opaque   2      26m
mysql-cluster01-passwd        Opaque   3      32m
mysql-cluster01-privsecrets   Opaque   2      26m
mysql-cluster01-router        Opaque   2      26m
mysql-user-pass               Opaque   4      32s
## 修改wordpress deploy编排资源文件
root@k8s-master1:~/learning-k8s/wordpress/wordpress# vi 03-deployment-wordpress.yam
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wordpress
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - image: wordpress:5.8-fpm
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql-cluster01
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-user-pass
              key: user.name
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-user-pass
              key: user.password
        - name: WORDPRESS_DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-user-pass
              key: database.name
        volumeMounts:
        - name: wordpress-app-data
          mountPath: /var/www/html/
      volumes:
      - name: wordpress-app-data
        persistentVolumeClaim:
          claimName: wordpress-app-data
## 部署wordpress
root@k8s-master1:~/learning-k8s/wordpress# kubectl apply -f wordpress/
## 查看资源创建状态
root@k8s-master1:~/learning-k8s/wordpress# kubectl get pod,svc 
NAME                                         READY   STATUS    RESTARTS   AGE
pod/mysql-cluster01-0                        2/2     Running   0          31m
pod/mysql-cluster01-1                        2/2     Running   0          31m
pod/mysql-cluster01-2                        2/2     Running   0          31m
pod/mysql-cluster01-router-8f6b88566-xp8gx   1/1     Running   0          24m
pod/wordpress-695749c74-26x2m                1/1     Running   0          71s

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                  AGE
service/kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                                                  23d
service/mysql-cluster01             ClusterIP   10.97.193.180    <none>        3306/TCP,33060/TCP,6446/TCP,6448/TCP,6447/TCP,6449/TCP   31m
service/mysql-cluster01-instances   ClusterIP   None             <none>        3306/TCP,33060/TCP,33061/TCP                             31m
service/wordpress                   ClusterIP   10.102.164.179   <none>        9000/TCP  
## 修改 nginx service 配置
root@k8s-master1:~/learning-k8s/wordpress# cd nginx/
root@k8s-master1:~/learning-k8s/wordpress/nginx# vi 02-service-nginx.yaml 
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - name: http-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: NodePort
  externalIPs:
  - 192.168.50.100
## 部署nginx
root@k8s-master1:~/learning-k8s/wordpress# kubectl apply -f nginx/
configmap/nginx-conf created
service/nginx created
deployment.apps/nginx created
## 查看部署的资源
root@k8s-master1:~/learning-k8s/wordpress# kubectl get pod,svc,cm
NAME                                         READY   STATUS    RESTARTS   AGE
pod/mysql-cluster01-0                        2/2     Running   0          39m
pod/mysql-cluster01-1                        2/2     Running   0          39m
pod/mysql-cluster01-2                        2/2     Running   0          39m
pod/mysql-cluster01-router-8f6b88566-xp8gx   1/1     Running   0          32m
pod/nginx-5b9c7b4c8f-2jdfk                   1/1     Running   0          31s
pod/wordpress-695749c74-26x2m                1/1     Running   0          9m16s

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP      PORT(S)                                                  AGE
service/kubernetes                  ClusterIP   10.96.0.1        <none>           443/TCP                                                  23d
service/mysql-cluster01             ClusterIP   10.97.193.180    <none>           3306/TCP,33060/TCP,6446/TCP,6448/TCP,6447/TCP,6449/TCP   39m
service/mysql-cluster01-instances   ClusterIP   None             <none>           3306/TCP,33060/TCP,33061/TCP                             39m
service/nginx                       NodePort    10.104.116.12    192.168.50.100   80:30147/TCP                                             31s
service/wordpress                   ClusterIP   10.102.164.179   <none>           9000/TCP                                                 9m16s

NAME                                 DATA   AGE
configmap/kube-root-ca.crt           1      23d
configmap/mysql-cluster01-initconf   8      39m
configmap/nginx-conf                 1      31s
```
### 1.2.6 访问wordpres
* 安装首页
![language](pictures/wp-select-language-02.png)
* 博客配置
![config](pictures/wp-setup-configuration-02.png)
* 访问首页
![access](pictures/wp-test-access-02.png)