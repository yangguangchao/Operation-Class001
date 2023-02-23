# 1. 添加用户
## 1.1 添加两个以上静态令牌认证的用户，例如tom和jerry，并认证到Kubernetes上
```bash
## 生成tom 和jerry 用户token
root@k8s-master1:~# echo "$(openssl rand -hex 3).$(openssl rand -hex 8)" 
e6f6fc.1bcbf217840be51b
root@k8s-master1:~# echo "$(openssl rand -hex 3).$(openssl rand -hex 8)" 
8946cb.63e2cb0b9289586e
root@k8s-master1:~# cd /etc/
## 备份kubernetes master节点配置,后面操作失败可以还原
root@k8s-master1:/etc# cp -pr kubernetes kubernetes.bak
## 创建token文件存放目录
root@k8s-master1:/etc/kubernetes# mkdir authfiles
root@k8s-master1:/etc/kubernetes# cd authfiles/
## 创建token文件
root@k8s-master1:/etc/kubernetes/authfiles# vi token.csv
e6f6fc.1bcbf217840be51b,tom,1001,kuberuser
8946cb.63e2cb0b9289586e,jerry,1002,kuberadmin
## 复制kube-apiserver资源编排文件到/tmp目录下编辑
root@k8s-master1:/etc/kubernetes/authfiles# cd ..
root@k8s-master1:/etc/kubernetes# ls
admin.conf  authfiles  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf
root@k8s-master1:/etc/kubernetes# cd manifests/
root@k8s-master1:/etc/kubernetes/manifests# cp kube-apiserver.yaml /tmp
## 编辑kube-apiserver资源编排文件
root@k8s-master1:/tmp# vi kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.50.201:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.50.201
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
## 开启token认证
    - --token-auth-file=/etc/kubernetes/authfiles/token.csv
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-issuer=https://kubernetes.default.svc.cluster.local
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: registry.aliyuncs.com/google_containers/kube-apiserver:v1.26.0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 192.168.50.201
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    name: kube-apiserver
    readinessProbe:
      failureThreshold: 3
      httpGet:
        host: 192.168.50.201
        path: /readyz
        port: 6443
        scheme: HTTPS
      periodSeconds: 1
      timeoutSeconds: 15
    resources:
      requests:
        cpu: 250m
    startupProbe:
      failureThreshold: 24
      httpGet:
        host: 192.168.50.201
        path: /livez
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 15
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/pki
      name: etc-pki
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
#挂载token认证的卷
    - mountPath: /etc/kubernetes/authfiles
      name: authfiles
      readOnly: true
  hostNetwork: true
  priorityClassName: system-node-critical
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/pki
      type: DirectoryOrCreate
    name: etc-pki
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
# 定义token认证要挂载的卷
  - hostPath:
      path: /etc/kubernetes/authfiles
      type: DirectoryOrCreate
    name: authfiles
status: {}
## 应用编辑后的kube-apiserver资源编排文件
root@k8s-master1:/tmp# cp kube-apiserver.yaml /etc/kubernetes/manifests/
## 查看资源失败，kube-apiserver正在重启
root@k8s-master1:/tmp# kubectl get pod
The connection to the server kubeapi.ygc.cn:6443 was refused - did you specify the right host or port?
## 等重启完成，再次查看资源
root@k8s-master1:/tmp# kubectl get pod
No resources found in default namespace.
## 登陆node1使用kubectl用tom和jerry的token进行验证
root@k8s-node1:~# kubectl --server https://kubeapi.ygc.cn:6443 --token="e6f6fc.1bcbf217840be51b" --certificate-authority=/etc/kubernetes/pki/ca.crt get pods
Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "default"
root@k8s-node1:~# kubectl --server https://kubeapi.ygc.cn:6443 --token="8946cb.63e2cb0b9289586e" --certificate-authority=/etc/kubernetes/pki/ca.crt get pods
Error from server (Forbidden): pods is forbidden: User "jerry" cannot list resource "pods" in API group "" in the namespace "default"
## 使用curl用tom和jerry的token进行验证
root@k8s-node1:~# curl -k -H "Authorization: Bearer e6f6fc.1bcbf217840be51b"  -k https://kubeapi.ygc.cn:6443/api/v1/namespaces/default/pods/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"tom\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
root@k8s-node1:~#curl -k -H "Authorization: Bearer 8946cb.63e2cb0b9289586e"  -k https://kubeapi.ygc.cn:6443/api/v1/namespaces/default/pods//
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User \"jerry\" cannot list resource \"pods\" in API group \"\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
## kubectl和curl验证都能识别到用户tom和jerry的信息,但是缺少授权信息,查看不到资源
```
## 1.2 添加两个以上的X509证书认证到Kubernetes的用户，比如mason和magedu
```bash
## 切换到/etc/kubernetes目录操作
root@k8s-master1:~# cd /etc/kubernetes
## 生成mason和magedu用户私钥
root@k8s-master1:/etc/kubernetes# umask 077; openssl genrsa -out ./pki/mason.key 4096
root@k8s-master1:/etc/kubernetes# umask 077; openssl genrsa -out ./pki/magedu.key 4096
## 创建mason和magedu用户证书签署请求
root@k8s-master1:/etc/kubernetes# openssl req -new -key ./pki/mason.key -out ./pki/mason.csr -subj "/CN=mason/O=developers"
root@k8s-master1:/etc/kubernetes# openssl req -new -key ./pki/magedu.key -out ./pki/magedu.csr -subj "/CN=magedu/O=operation"
## 由Kubernetes CA签署证书
root@k8s-master1:/etc/kubernetes# openssl x509 -req -days 3650 -CA ./pki/ca.crt  -CAkey ./pki/ca.key -CAcreateserial  -in ./pki/mason.csr -out ./pki/mason.crt
Certificate request self-signature ok
subject=CN = mason, O = developers
root@k8s-master1:/etc/kubernetes# openssl x509 -req -days 3650 -CA ./pki/ca.crt  -CAkey ./pki/ca.key -CAcreateserial  -in ./pki/magedu.csr -out ./pki/magedu.crt
Certificate request self-signature ok
subject=CN = magedu, O = operation
## 复制mason和magedu的crt和key文件到k8s-node1
root@k8s-master1:/etc/kubernetes# scp -rp ./pki/{mason.crt,mason.key,magedu.crt,magedu.key}  k8s-node1:/etc/kubernetes/pki
The authenticity of host 'k8s-node1 (192.168.50.101)' can't be established.
ED25519 key fingerprint is SHA256:3q6sbtQEAnteY3H4acgJK3xobJdxCrBUIFFvLDbZESU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'k8s-node1' (ED25519) to the list of known hosts.
root@k8s-node1's password: 
mason.crt                                                                                                      100% 1363     1.5MB/s   00:00    
mason.key                                                                                                      100% 3272     3.8MB/s   00:00    
magedu.crt                                                                                                     100% 1363     2.1MB/s   00:00    
magedu.key                                                                                                     100% 3272     4.4MB/s   00:00 
## 在k8s-node01上发起访问测试
root@k8s-node1:~# kubectl -s https://kubeapi.ygc.cn:6443 --client-certificate=/etc/kubernetes/pki/mason.crt --client-key=/etc/kubernetes/pki/mason.key --certificate-authority=/etc/kubernetes/pki/ca.crt get pods
Error from server (Forbidden): pods is forbidden: User "mason" cannot list resource "pods" in API group "" in the namespace "default"
root@k8s-node1:~# kubectl -s https://kubeapi.ygc.cn:6443 --client-certificate=/etc/kubernetes/pki/magedu.crt --client-key=/etc/kubernetes/pki/magedu.key --certificate-authority=/etc/kubernetes/pki/ca.crt get pods
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list resource "pods" in API group "" in the namespace "default"
## kubectl验证能识别到用户mason和magedu的信息,但是缺少授权信息,查看不到资源
```
## 1.3 把认证凭据添加到kubeconfig配置文件进行加载
### 1.3.1 把静态令牌认证的用户添加到kubeconfig配置文件
```bash
## 定义Cluster,提供包括集群名称、API Server URL和信任的CA的证书相关的配置；clusters配置段中的各列表项名称需要惟
root@k8s-master1:~# kubectl config set-cluster kube-test --embed-certs=true --certificate-authority=/etc/kubernetes/pki/ca.crt --server="https://kubeapi.ygc.cn:6443" --kubeconfig=$HOME/.kube/kubeusers.conf
Cluster "kube-test" set.
## 定义User,添加身份凭据,使用静态令牌文件认证的客户端提供令牌令牌即可
root@k8s-master1:~# kubectl config set-credentials tom --token="e6f6fc.1bcbf217840be51b"  --kubeconfig=$HOME/.kube/kubeusers.conf
User "tom" set.
root@k8s-master1:~# kubectl config set-credentials jerry --token="8946cb.63e2cb0b9289586e"  --kubeconfig=$HOME/.kube/kubeusers.conf
User "jerry" set.
## 定义Context,为用户tom和jerry的身份凭据与kube-test集群建立映射关系
root@k8s-master1:~# kubectl config set-context tom@kube-test --cluster=kube-test --user=tom --kubeconfig=$HOME/.kube/kubeusers.conf
Context "tom@kube-test" created.
root@k8s-master1:~# kubectl config set-context jerry@kube-test --cluster=kube-test --user=jerry --kubeconfig=$HOME/.kube/kubeusers.conf
Context "jerry@kube-test" created.
## 验证kubeconfig文件配置
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf --context=tom@kube-test
Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "default"
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf --context=jerry@kube-test
Error from server (Forbidden): pods is forbidden: User "jerry" cannot list resource "pods" in API group "" in the namespace "default"
## 设定Current-Context
root@k8s-master1:~# kubectl config use-context tom@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "tom@kube-test".
## 验证Current-Context配置
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf
Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "default"
## 切换为用户jerry的Current-Context并验证
root@k8s-master1:~# kubectl config use-context jerry@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "jerry@kube-test".
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf
Error from server (Forbidden): pods is forbidden: User "jerry" cannot list resource "pods" in API group "" in the namespace "default"
```
### 1.3.2 将基于X509客户端证书认证的用户添加至kubeusers.conf文件中
```bash
## 定义Cluster,使用不同的身份凭据访问同一集群时,集群相关的配置无须重复定义
## 定义User,添加身份凭据,基于X509客户端证书认证时,需要提供客户端证书和私钥
root@k8s-master1:~# kubectl config set-credentials mason --embed-certs=true --client-certificate=/etc/kubernetes/pki/mason.crt --client-key=/etc/kubernetes/pki/mason.key --kubeconfig=$HOME/.kube/kubeusers.conf
User "mason" set.
root@k8s-master1:~# kubectl config set-credentials magedu --embed-certs=true --client-certificate=/etc/kubernetes/pki/magedu.crt --client-key=/etc/kubernetes/pki/magedu.key --kubeconfig=$HOME/.kube/kubeusers.conf
User "magedu" set.
## 定义Context,为用户mason和magedu的身份凭据与kube-test集群建立映射关系
root@k8s-master1:~# kubectl config set-context mason@kube-test --cluster=kube-test --user=mason --kubeconfig=$HOME/.kube/kubeusers.conf
Context "mason@kube-test" created.
root@k8s-master1:~# kubectl config set-context magedu@kube-test --cluster=kube-test --user=magedu --kubeconfig=$HOME/.kube/kubeusers.conf
Context "magedu@kube-test" created.
## 验证基于x509证书的kuberconfig用户配置
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf --context=mason@kube-test
Error from server (Forbidden): pods is forbidden: User "mason" cannot list resource "pods" in API group "" in the namespace "default"
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf --context=magedu@kube-test
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list resource "pods" in API group "" in the namespace "default"
## 设定Current-Context
root@k8s-master1:~# kubectl config use-context mason@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "mason@kube-test".
## 验证Current-Context
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf
Error from server (Forbidden): pods is forbidden: User "mason" cannot list resource "pods" in API group "" in the namespace "default"
## 切换Current-Context为magedu用户配置，并验证
root@k8s-master1:~# kubectl config use-context magedu@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "magedu@kube-test".
root@k8s-master1:~# kubectl get pod --kubeconfig=$HOME/.kube/kubeusers.conf
Error from server (Forbidden): pods is forbidden: User "magedu" cannot list resource "pods" in API group "" in the namespace "default"
```
# 2. 使用资源配置文件创建ServiceAccount，并附加一个imagePullSecrets
```bash
## 创建test namespace
root@k8s-master1:~# k create namespace test
namespace/test created
## 创建docker-registry secret
root@k8s-master1:~# k create secret docker-registry mydcokrhub --docker-username=USER --docker-password=PASSWORD -n test
secret/mydcokrhub created
## 编辑一个ServiceAccount配置文件
root@k8s-master1:~# vi mysa-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysa
  namespace: test
## 创建serviceaccount
root@k8s-master1:~# k apply -f mysa-serviceaccount.yaml 
serviceaccount/mysa created
## 查看serviceaccount
root@k8s-master1:~# k get serviceaccounts -n test
NAME      SECRETS   AGE
default   0         5m54s
mysa      0         10s
## 在test命名空间查创建一个nginx，使用docker hub中的私有镜像
root@k8s-master1:~# k create deployment nginx --image=yang5188/nginx:latest -n test
deployment.apps/nginx created
## 查看pod,发现拉取镜像失败
root@k8s-master1:~# k get pod -n test
NAME                     READY   STATUS             RESTARTS   AGE
nginx-6b44bd68f4-ll7g6   0/1     ImagePullBackOff   0          64s
## 查看pod事件,提示没有权限拉取镜像,需要登陆
root@k8s-master1:~# k describe pod nginx-6b44bd68f4-ll7g6 -n test
Name:             nginx-6b44bd68f4-ll7g6
Namespace:        test
Priority:         0
Service Account:  default
Node:             k8s-node1/192.168.50.101
Start Time:       Thu, 23 Feb 2023 22:38:29 +0800
Labels:           app=nginx
                  pod-template-hash=6b44bd68f4
Annotations:      <none>
Status:           Pending
IP:               10.244.1.21
IPs:
  IP:           10.244.1.21
Controlled By:  ReplicaSet/nginx-6b44bd68f4
Containers:
  nginx:
    Container ID:   
    Image:          yang5188/nginx:latest
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hh62c (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-hh62c:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  85s                default-scheduler  Successfully assigned test/nginx-6b44bd68f4-ll7g6 to k8s-node1
  Normal   Pulling    34s (x3 over 84s)  kubelet            Pulling image "yang5188/nginx:latest"
  Warning  Failed     29s (x3 over 79s)  kubelet            Failed to pull image "yang5188/nginx:latest": rpc error: code = Unknown desc = Error response from daemon: pull access denied for yang5188/nginx, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     29s (x3 over 79s)  kubelet            Error: ErrImagePull
  Normal   BackOff    2s (x4 over 78s)   kubelet            Back-off pulling image "yang5188/nginx:latest"
  Warning  Failed     2s (x4 over 78s)   kubelet            Error: ImagePullBackOff
## 编辑serviceaccount资源文件，加入imagePullSecret
root@k8s-master1:~# vi mysa-serviceaccount.yaml 
apiVersion: v1
kind: ServiceAccount
imagePullSecrets:
- name: mydcokrhub
metadata:
  name: mysa
  namespace: test
## 重新应用yaml文件
root@k8s-master1:~# k apply -f mysa-serviceaccount.yaml 
serviceaccount/mysa configured
## 修改nginx serviceAccountName
root@k8s-master1:~# k edit deploy nginx -n test
deployment.apps/nginx edited
    spec:
      containers:
      - image: yang5188/nginx:latest
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      #使用创建serviceaccount mysa
      serviceAccountName: mysa
      schedulerName: default-scheduler
## 查看pod,使用了加入imagePullSecret serviceaccount,启动成功
root@k8s-master1:~# k get pod -n test
NAME                     READY   STATUS    RESTARTS   AGE
nginx-76fd449d87-65jlh   1/1     Running   0          35s
```
# 3. 用户授权
## 3.1 为tom用户授予管理blog名称空间的权限
```bash
## 创建 blog namespace
root@k8s-master1:~# k create namespace blog
namespace/blog created
## 创建nginx deployment控制器应用
root@k8s-master1:~# k create deployment nginx --image=nginx -n blog
deployment.apps/nginx created
## 创建nginx service
root@k8s-master1:~# k create service clusterip nginx --tcp=80:80 -n blog
service/nginx created
## 查看创建的资源
root@k8s-master1:~# k get pod,svc -n blog
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-748c667d99-7t4r7   1/1     Running   0          3m14s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.98.56.126   <none>        80/TCP    84s
## 创建clusterrole，允许管理blog命名空间
root@k8s-master1:~# kubectl create clusterrole manager-blog-ns --verb=get,list,create,watch,patch,update,delete,deletecollection --resource=namespaces --resource-name=blog
clusterrole.rbac.authorization.k8s.io/manager-blog-ns created
## 为tom用户授权
root@k8s-master1:~# kubectl create clusterrolebinding tom-as-manage-blog-ns --clusterrole=manager-blog-ns --user=tom
clusterrolebinding.rbac.authorization.k8s.io/tom-as-manage-blog-ns created
##切换为用户tom的Current-Context并验证
root@k8s-master1:~# kubectl config use-context tom@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "tom@kube-test".
##查看blog namaspace 
root@k8s-master1:~# k get namespace blog --kubeconfig=$HOME/.kube/kubeusers.conf
NAME   STATUS   AGE
blog   Active   14m
## 查看所有namespace创建的资源，提示没权限
root@k8s-master1:~# k get pod,svc -n blog --kubeconfig=$HOME/.kube/kubeusers.conf
Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "blog"
Error from server (Forbidden): services is forbidden: User "tom" cannot list resource "services" in API group "" in the namespace "blog"
## 删除blog namespace,可以正常删除
root@k8s-master1:~# k delete namespace blog --kubeconfig=$HOME/.kube/kubeusers.conf
namespace "blog" deleted
```
## 3.2 为jerry授予管理整个集群的权限
```bash
## 为jerry授予管理集群的权限
root@k8s-master1:~# kubectl create clusterrolebinding jerry--as-cluster-admin --clusterrole=cluster-admin --user=jerry
clusterrolebinding.rbac.authorization.k8s.io/jerry--as-cluster-admin created
##切换为用户jerry的Current-Context并验证
root@k8s-master1:~# kubectl config use-context jerry@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "jerry@kube-test".
## 查看pod资源
root@k8s-master1:~# k get pod -A --kubeconfig=$HOME/.kube/kubeusers.conf
NAMESPACE        NAME                                  READY   STATUS    RESTARTS      AGE
kube-flannel     kube-flannel-ds-8k6tc                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-f79f7                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-v5dgm                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-zfc9q                 1/1     Running   2 (24h ago)   30d
kube-system      coredns-5bbd96d687-td5rp              1/1     Running   2 (24h ago)   30d
kube-system      coredns-5bbd96d687-znblb              1/1     Running   2 (24h ago)   30d
kube-system      csi-nfs-controller-5497c9c89b-fvlv8   3/3     Running   4 (23h ago)   6d23h
kube-system      csi-nfs-node-7xv27                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-jqfhp                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-kxqdc                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-x4gnm                    3/3     Running   3 (24h ago)   6d23h
kube-system      etcd-k8s-master1                      1/1     Running   2 (24h ago)   30d
kube-system      kube-apiserver-k8s-master1            1/1     Running   0             23h
kube-system      kube-controller-manager-k8s-master1   1/1     Running   3 (23h ago)   30d
kube-system      kube-proxy-bjzbj                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-jk5v9                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-mdrtj                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-x7gvp                      1/1     Running   2 (24h ago)   30d
kube-system      kube-scheduler-k8s-master1            1/1     Running   3 (23h ago)   30d
mysql-operator   mysql-operator-cc9796cb8-f69lf        1/1     Running   1 (24h ago)   6d23h
nfs              nfs-server-857f859f57-rr267           1/1     Running   1 (24h ago)   6d23h
## 查看service资源
root@k8s-master1:~# k get svc -A --kubeconfig=$HOME/.kube/kubeusers.conf
NAMESPACE        NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default          kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP                  30d
kube-system      kube-dns         ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   30d
mysql-operator   mysql-operator   ClusterIP   10.102.120.88   <none>        9443/TCP                 6d23h
nfs              nfs-server       ClusterIP   10.102.183.67   <none>        2049/TCP,111/UDP         6d23h
## 创建namespace blog,可以成功创建
root@k8s-master1:~# k create namespace blog --kubeconfig=$HOME/.kube/kubeusers.conf
namespace/blog created
## 创建deployment 应用nginx可以成功创建
root@k8s-master1:~# k create deploy nginx --image=nginx --kubeconfig=$HOME/.kube/kubeusers.conf
deployment.apps/nginx created
```
## 3.3 为mason用户授予读取集群资源的权限
```bash
## 为mason用户授予读取集群资源的权限
root@k8s-master1:~# kubectl create clusterrolebinding mason--as-view --clusterrole=view --user=mason
clusterrolebinding.rbac.authorization.k8s.io/mason--as-view created
##切换为用户mason的Current-Context并验证
root@k8s-master1:~# kubectl config use-context mason@kube-test --kubeconfig=$HOME/.kube/kubeusers.conf
Switched to context "mason@kube-test".
## 查看所有pdo资源，可以正常查看
root@k8s-master1:~# k get pod -A --kubeconfig=$HOME/.kube/kubeusers.conf
NAMESPACE        NAME                                  READY   STATUS    RESTARTS      AGE
default          nginx-748c667d99-7s42t                1/1     Running   0             3m47s
kube-flannel     kube-flannel-ds-8k6tc                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-f79f7                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-v5dgm                 1/1     Running   2 (24h ago)   30d
kube-flannel     kube-flannel-ds-zfc9q                 1/1     Running   2 (24h ago)   30d
kube-system      coredns-5bbd96d687-td5rp              1/1     Running   2 (24h ago)   30d
kube-system      coredns-5bbd96d687-znblb              1/1     Running   2 (24h ago)   30d
kube-system      csi-nfs-controller-5497c9c89b-fvlv8   3/3     Running   4 (24h ago)   6d23h
kube-system      csi-nfs-node-7xv27                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-jqfhp                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-kxqdc                    3/3     Running   3 (24h ago)   6d23h
kube-system      csi-nfs-node-x4gnm                    3/3     Running   3 (24h ago)   6d23h
kube-system      etcd-k8s-master1                      1/1     Running   2 (24h ago)   30d
kube-system      kube-apiserver-k8s-master1            1/1     Running   0             24h
kube-system      kube-controller-manager-k8s-master1   1/1     Running   3 (24h ago)   30d
kube-system      kube-proxy-bjzbj                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-jk5v9                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-mdrtj                      1/1     Running   2 (24h ago)   30d
kube-system      kube-proxy-x7gvp                      1/1     Running   2 (24h ago)   30d
kube-system      kube-scheduler-k8s-master1            1/1     Running   3 (24h ago)   30d
mysql-operator   mysql-operator-cc9796cb8-f69lf        1/1     Running   1 (24h ago)   6d23h
nfs              nfs-server-857f859f57-rr267           1/1     Running   1 (24h ago)   6d23h
## 创建一个deployment nginx应用,提示mason用户没权限
root@k8s-master1:~# k create deployment nginx --image=nginx --kubeconfig=$HOME/.kube/kubeusers.conf
error: failed to create deployment: deployments.apps is forbidden: User "mason" cannot create resource "deployments" in API group "apps" in the namespace "default"
```
# 4. 部署jenkins和prometheus
## 4.1 部署Jenkins、Prometheus-Server、Node-Exporter至Kubernetes集群
### 4.1.1 部署Jenkins至Kubernetes集群
```bash
## 部署jenkins
root@k8s-master1:~# cd learning-k8s/jenkins/deploy/
root@k8s-master1:~/learning-k8s/jenkins/deploy# k apply -f 01-namespace-jenkins.yaml -f 02-pvc-jenkins.yaml -f 03-rbac-jenkins.yaml -f 04-deploy-jenkins.yaml -f 05-service-jenkins.yaml -f 06-pvc-maven-cache.yaml
namespace/jenkins created
persistentvolumeclaim/jenkins-pvc created
serviceaccount/jenkins-master created
clusterrole.rbac.authorization.k8s.io/jenkins-master created
clusterrolebinding.rbac.authorization.k8s.io/jenkins-master created
deployment.apps/jenkins created
service/jenkins created
service/jenkins-jnlp created
persistentvolumeclaim/pvc-maven-cache created
## 查看创建的资源
root@k8s-master1:~/learning-k8s/jenkins/deploy# k get pod,svc,pvc,sa -n jenkins
NAME                           READY   STATUS    RESTARTS   AGE
pod/jenkins-6c5f8665c8-wzcs5   1/1     Running   0          2m10s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/jenkins        NodePort    10.103.57.249    <none>        8080:30413/TCP   2m10s
service/jenkins-jnlp   ClusterIP   10.100.109.249   <none>        50000/TCP        2m10s

NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/jenkins-pvc       Bound    pvc-215eb65c-04d6-4b5c-a3a1-a9f2cd986075   10Gi       RWX            nfs-csi        2m11s
persistentvolumeclaim/pvc-maven-cache   Bound    pvc-bb57cb5e-6b81-4e5c-8221-e1bf6a066a1c   10Gi       RWX            nfs-csi        2m10s

NAME                            SECRETS   AGE
serviceaccount/default          0         2m11s
serviceaccount/jenkins-master   0         2m11s
## 查看jenkins密码
root@k8s-master1:~/learning-k8s/jenkins/deploy# k get pod -n jenkins 
NAME                       READY   STATUS    RESTARTS   AGE
jenkins-6c5f8665c8-wzcs5   1/1     Running   0          4m46s
root@k8s-master1:~/learning-k8s/jenkins/deploy# k logs jenkins-6c5f8665c8-wzcs5 -n jenkins
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

a8010c7580024f998e31b2d2a5fa3206

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```
* 浏览器访问jenkins，输入密码
![passwd](pictures/jenkins-setup-default-passwd-01.png)
* 选择安装推荐的插件
![plugin](pictures/jenkins-install-plugin-01.png)
* 设置管理员账号
![admin](pictures/jenkins-setup-admin-01.png)
* 安装完成
![success](pictures/jenkins-install-success-01.png)
* 安装kubernetes插件
![kubernetes](pictures/jenkins-install-plugin-kubernetes-01.png)
* 系统管理-->节点管理-->configureClouds
![config](pictures/jenkins-setup-kubernetes-configuration-02.png)
* 设置jenkins URL
![url](pictures/jenkins-setup-url-01.png)
* 创建一个freestype-job-001任务
![job](pictures/jenkins-create-job-freestyle-job01.png)
* 查看kubernetes jenkins任务pod
```bahs
root@k8s-master1:~/learning-k8s/jenkins/deploy# k get pod -n jenkins -w
NAME                       READY   STATUS    RESTARTS      AGE
jenkins-6c5f8665c8-wzcs5   1/1     Running   1 (50m ago)   69m
kube-agent-15qbw           0/1     Pending   0             0s
kube-agent-15qbw           0/1     Pending   0             0s
kube-agent-15qbw           0/1     ContainerCreating   0             0s
kube-agent-15qbw           1/1     Running             0             2s
kube-agent-15qbw           1/1     Terminating         0             9s
kube-agent-15qbw           0/1     Terminating         0             10s
kube-agent-15qbw           0/1     Terminating         0             10s
kube-agent-15qbw           0/1     Terminating         0             10s

```
### 4.1.2 部署Prometheus-Server、Node-Exporter至Kubernetes集群
```bash
## 下载实例代码
root@k8s-master1:~# git clone https://github.com/iKubernetes/k8s-prom.git
Cloning into 'k8s-prom'...
remote: Enumerating objects: 89, done.
remote: Counting objects: 100% (40/40), done.
remote: Compressing objects: 100% (29/29), done.
remote: Total 89 (delta 18), reused 29 (delta 11), pack-reused 49
Receiving objects: 100% (89/89), 20.78 KiB | 1.15 MiB/s, done.
Resolving deltas: 100% (29/29), done.
## 部署prometheus
root@k8s-master1:~/k8s-prom# k apply -f namespace.yaml 
namespace/prom created
root@k8s-master1:~/k8s-prom# k apply -f prometheus
configmap/prometheus-config created
deployment.apps/prometheus-server created
clusterrole.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
## 查看创建的资源
root@k8s-master1:~/k8s-prom# k get pod,svc,sa,cm -n prom
NAME                                    READY   STATUS    RESTARTS   AGE
pod/prometheus-server-8f7bc69b5-r6kbl   1/1     Running   0          64s

NAME                 TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/prometheus   NodePort   10.97.42.232   <none>        9090:30090/TCP   64s

NAME                        SECRETS   AGE
serviceaccount/default      0         77s
serviceaccount/prometheus   0         64s

NAME                          DATA   AGE
configmap/kube-root-ca.crt    1      77s
configmap/prometheus-config   1      64s
## 部署node-exporter
root@k8s-master1:~/k8s-prom# k apply -f node_exporter
daemonset.apps/prometheus-node-exporter created
service/prometheus-node-exporter created
```
* 查看prometheus
![targets](pictures/show-prometheus-targets-01.png)
## 4.2 使用Ingress开放至集群外部，Jenkins要使用https协议开放；
### 4.2.1 部署ingress
```bash
## 下载ingress资源编排文件
root@k8s-master1:~# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml
## 更换镜像地址
root@k8s-master1:~# vi deploy.yaml 
registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343 --> yang5188/kube-webhook-certgen:v20220916-gd32f8c343
registry.k8s.io/ingress-nginx/controller:v1.6.4 --> registry.aliyuncs.com/google_containers/nginx-ingress-controller:v1.6.4
## 部署ingress-controller
root@k8s-master1:~# k apply -f deploy.yaml 
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
## 检查部署情况
root@k8s-master1:~# kubectl get pods --namespace=ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-mzznk        0/1     Completed   0          11m
ingress-nginx-admission-patch-qhlqb         0/1     Completed   0          11m
ingress-nginx-controller-66457b7747-jj2z9   1/1     Running     0          6m49s
## 修改ingress-nginx-controller service配置
root@k8s-master1:~# k edit svc ingress-nginx-controller -n ingress-nginx
  externalTrafficPolicy: Cluster
  externalIPs: 192.168.50.100
```
* 测试访问
![access](pictures/access-nginx-01.png)
# 5. helm部署服务
## 5.1 使用helm部署主从复制的MySQL集群，部署wordpress，并使用ingress暴露到集群外部
## 5.2 使用helm部署harbor，成功验证推送Image至Harbor上；使用helm部署一个redis cluster至Kubernetes上；