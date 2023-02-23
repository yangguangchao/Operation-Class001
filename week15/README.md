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
k create serviceaccount mysa -n test
k create secret docker-registry mydcokrhub --docker-username=yang5188 --docker-password=ygc19910723@ -n test

```
# 3. 为tom用户授予管理blog名称空间的权限；为jerry授予管理整个集群的权限；为mason用户授予读取集群资源的权限；
```bash
使用rclusterrolebinding
kubectl create clusterrole manager-blog-ns --verb=get,list,create,wathch,patch,update,delete,deletecollection --resource=namespaces --resource-name=blog
kubectl create clusterrolebinding tom-as-manage-blog-ns --clusterrole=manager-blog-ns --user=tom

使用rolebinding
kubectl create clusterrole manager-blog-ns --verb=get,list,create,wathch,patch,update,delete,deletecollection --resource=namespaces
kubectl create rolebinding tom-as-manager-blog-ns --clusterrole=manager-blog-ns --user=tom -n blog


kubectl create clusterrolebinding jerry--as-cluster-admin --clusterrole=cluster-admin --user=jerry
kubectl create clusterrolebinding mason--as-view --clusterrole=view --user=mason
```
# 4. 部署Jenkins、Prometheus-Server、Node-Exporter至Kubernetes集群；而后使用Ingress开放至集群外部，Jenkins要使用https协议开放；
# 5. helm部署服务
## 5.1 使用helm部署主从复制的MySQL集群，部署wordpress，并使用ingress暴露到集群外部
## 5.2 使用helm部署harbor，成功验证推送Image至Harbor上；使用helm部署一个redis cluster至Kubernetes上；