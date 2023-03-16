# 1. 在 K8s 环境基于 daemonset 部署日志收集组件实现 pod 日志收集
# 2. 在 K8s 环境对 pod 添加 sidecar 容器实现业务日志收集
# 3. 在 K8s 环境容器中启动日志收集进程实现业务日志收集
# 4. 通过 prometheus 对 CoreDNS 进行监控并在 grafana 显示监控图形
# 5. 对 K8s 集群进行 master 节点扩容、node 节点扩容
# 6. 对 K8s 集群进行小版本升级
# 7. 基于 ceph rbd 及 cephfs 持久化 K8s 中 pod 的业务数据
## 7.1 创建初始化rbd
```bash
## 创建新的rbd
bash-4.4$ ceph osd pool create ygc-rbd-pool1 32 32
pool 'ygc-rbd-pool1' created
## 验证存储池
bash-4.4$ ceph osd pool ls
.mgr
replicapool
myfs-metadata
myfs-replicated
ygc-rbd-pool1
## 存储池启用rbd
bash-4.4$ ceph osd pool application enable ygc-rbd-pool1 rbd
enabled application 'rbd' on pool 'ygc-rbd-pool1'
## 初始化rbd
bash-4.4$ rbd pool init -p ygc-rbd-pool1
```
## 7.2 创建image
```bash
## 创建镜像
bash-4.4$ rbd create ygc-img-img1 --size 5G --pool ygc-rbd-pool1 --image-format 2 --image-feature layering
## 验证镜像
bash-4.4$ rbd ls --pool ygc-rbd-pool1
ygc-img-img1
## 验证镜像信息
bash-4.4$ rbd --image ygc-img-img1 --pool ygc-rbd-pool1 info
rbd image 'ygc-img-img1':
	size 5 GiB in 1280 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 68bf27d7961e
	block_name_prefix: rbd_data.68bf27d7961e
	format: 2
	features: layering
	op_features: 
	flags: 
	create_timestamp: Thu Mar 16 13:34:29 2023
	access_timestamp: Thu Mar 16 13:34:29 2023
	modify_timestamp: Thu Mar 16 13:34:29 2023
```
## 7.3 客户端安装ceph-common
```bash
## 验证ceph 版本
root@k8s-master1:~# apt-cache madison ceph-common
## 各master 与node 节点配置apt 源
root@k8s-master1:~# apt install ceph-common -y
```
## 7.4 创建ceph用户与授权
```bash
## 创建用户并授权
bash-4.4$ ceph auth get-or-create client.chaoge-ygc mon 'allow r' osd 'allow * pool=ygc-rbd-pool1'
[client.chaoge-ygc]
	key = AQAkJBNkFaQ1EBAAIGlG57NrXmlNh1ho+iKunA==
## 验证用户
[client.chaoge-ygc]
	key = AQAkJBNkFaQ1EBAAIGlG57NrXmlNh1ho+iKunA==
	caps mon = "allow r"
	caps osd = "allow * pool=ygc-rbd-pool1"
exported keyring for client.chaoge-ygc
## 导出用户信息至keyring 文件
bash-4.4$ ceph auth get client.chaoge-ygc -o ceph.client.chaoge-ygc.keyring
exported keyring for client.chaoge-ygc

## 拷贝keyring到master节点
root@k8s-master1:~#  kubectl cp rook-ceph/rook-ceph-tools-7c4b8bb9b5-dr7w8:/tmp/ceph.client.chaoge-ygc.keyring ./ceph.client.chaoge-ygc.keyring
## 同步认证文件到k8s 各master 及node 节点
root@k8s-master1:~# cp ceph.client.chaoge-ygc.keyring /etc/ceph/
root@k8s-master1:~# scp ceph.client.chaoge-ygc.keyring 172.31.2.6:/etc/ceph/
root@k8s-master1:~# scp ceph.client.chaoge-ygc.keyring 172.31.2.7:/etc/ceph/
root@k8s-master1:~# scp ceph.client.chaoge-ygc.keyring 172.31.2.8:/etc/ceph/
## 拷贝配置文件到master节点
root@k8s-master1:~# kubectl cp rook-ceph/rook-ceph-tools-7c4b8bb9b5-dr7w8:/etc/ceph/ceph.conf ./ceph.conf
## 同步配置文件到k8s 各master 及node 节点
root@k8s-master1:~# cp ceph.conf /etc/ceph/
root@k8s-master1:~# scp ceph.conf 172.31.2.6:/etc/ceph/
root@k8s-master1:~# scp ceph.conf 172.31.2.7:/etc/ceph/
root@k8s-master1:~# scp ceph.conf 172.31.2.8:/etc/ceph/
## 在k8s node 节点验证用户权限
root@k8s-node1:~# ceph --user chaoge-ygc -s
  cluster:
    id:     b14c452b-3406-4f96-85e1-7556e931a1e8
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum a,b,c (age 5h)
    mgr: b(active, since 3h), standbys: a
    mds: 1/1 daemons up, 1 hot standby
    osd: 3 osds: 3 up (since 5h), 3 in (since 5h)
 
  data:
    volumes: 1/1 healthy
    pools:   5 pools, 129 pgs
    objects: 43 objects, 3.7 MiB
    usage:   84 MiB used, 3.0 TiB / 3 TiB avail
    pgs:     129 active+clean
 
  io:
    client:   1.2 KiB/s rd, 2 op/s rd, 0 op/s wr
## 验证镜像访问权限
root@k8s-node1:~# rbd --id chaoge-ygc ls --pool=ygc-rbd-pool1
ygc-img-img1
```
## 7.4 通过keyring文件挂载rbd
### 7.4.1 通过keyring文件直接挂载-busybox
```bash
## pod yaml 文件
root@k8s-master1:~/ceph-case# cat case1-busybox-keyring.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - image: busybox 
    command:
      - sleep
      - "3600"
    imagePullPolicy: Always 
    name: busybox
    #restartPolicy: Always
    volumeMounts:
    - name: rbd-data1
      mountPath: /data
  volumes:
    - name: rbd-data1
      rbd:
        monitors:
        - '10.100.135.254:6789'
        - '10.100.100.35:6789'
        - '10.100.232.104:6789'
        pool: ygc-rbd-pool1
        image: ygc-img-img1
        fsType: ext4
        readOnly: false
        user: chaoge-ygc
        keyring: /etc/ceph/ceph.client.chaoge-ygc.keyring
## 创建pod
root@k8s-master1:~/ceph-case# kubectl apply -f case1-busybox-keyring.yaml 
## 查看pod
root@k8s-master1:~/ceph-case# kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          41s
## 到pod 验证rbd 是否挂载成功
root@k8s-master1:~/ceph-case# kubectl exec -it busybox sh
/ # df 
Filesystem           1K-blocks      Used Available Use% Mounted on
overlay              100820384  11215096  84965108  12% /
tmpfs                    65536         0     65536   0% /dev
/dev/rbd0              5074592        24   5058184   0% /data
/dev/mapper/ubuntu--vg-ubuntu--lv
                     100820384  11215096  84965108  12% /dev/termination-log
/dev/mapper/ubuntu--vg-ubuntu--lv
                     100820384  11215096  84965108  12% /etc/resolv.conf
/dev/mapper/ubuntu--vg-ubuntu--lv
                     100820384  11215096  84965108  12% /etc/hostname
/dev/mapper/ubuntu--vg-ubuntu--lv
                     100820384  11215096  84965108  12% /etc/hosts
shm                      65536         0     65536   0% /dev/shm
tmpfs                  3880788        12   3880776   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                  1991592         0   1991592   0% /proc/acpi
tmpfs                    65536         0     65536   0% /proc/kcore
tmpfs                    65536         0     65536   0% /proc/keys
tmpfs                    65536         0     65536   0% /proc/timer_list
tmpfs                  1991592         0   1991592   0% /proc/scsi
tmpfs                  1991592         0   1991592   0% /sys/firmware

```
### 7.4.2 通过keyring文件直接挂载-nginx
```bash
## pod yaml 文件
root@k8s-master1:~/ceph-case# cat case2-1-nginx-keyring-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        #image: mysql:5.6.46 
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: magedu123456
        ports:
        - containerPort: 80

        volumeMounts:
        - name: rbd-data1
          mountPath: /usr/share/nginx/html/jike
          #mountPath: /var/lib/mysql
      volumes:
        - name: rbd-data1
          rbd:
            monitors:
            - '10.100.135.254:6789'
            - '10.100.100.35:6789'
            - '10.100.232.104:6789'
            pool: ygc-rbd-pool1
            image: ygc-img-img1
            fsType: ext4
            readOnly: false
            user: chaoge-ygc
            keyring: /etc/ceph/ceph.client.chaoge-ygc.keyring
## 创建pod
root@k8s-master1:~/ceph-case# kubectl apply -f case2-1-nginx-keyring-deployment.yaml 
deployment.apps/nginx-deployment created
## 查看pod
root@k8s-master1:~/ceph-case# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-796bf79987-7mtk5   1/1     Running   0          59s
## 到pod 验证rbd 是否挂载成功
root@k8s-master1:~/ceph-case# kubectl exec -it nginx-deployment-796bf79987-7mtk5 -- bash
root@nginx-deployment-796bf79987-7mtk5:/# df -h
Filesystem                         Size  Used Avail Use% Mounted on
overlay                             97G   11G   82G  12% /
tmpfs                               64M     0   64M   0% /dev
/dev/mapper/ubuntu--vg-ubuntu--lv   97G   11G   82G  12% /etc/hosts
shm                                 64M     0   64M   0% /dev/shm
/dev/rbd0                          4.9G   24K  4.9G   1% /usr/share/nginx/html/jike
tmpfs                              3.8G   12K  3.8G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                              1.9G     0  1.9G   0% /proc/acpi
tmpfs                              1.9G     0  1.9G   0% /proc/scsi
tmpfs                              1.9G     0  1.9G   0% /sys/firmware
## 在挂载目录下写入一个html文件 
root@nginx-deployment-796bf79987-7mtk5:/# cd /usr/share/nginx/html/jike/
root@nginx-deployment-796bf79987-7mtk5:/usr/share/nginx/html/jike# echo "<h1>welcome to chaoge home</h1>" >> index.html
##service 文件
root@k8s-master1:~/ceph-case# cat case2-2-nginx-service.yaml 
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: ng-deploy-80-label
  name: ng-deploy-80
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30081
  selector:
    app: ng-deploy-80
## 创建service
root@k8s-master1:~/ceph-case# vi case2-2-nginx-service.yaml 
root@k8s-master1:~/ceph-case# kubectl apply -f case2-2-nginx-service.yaml 
service/ng-deploy-80 created
## 查看service
root@k8s-master1:~/ceph-case# kubectl get svc
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP   10.100.0.1      <none>        443/TCP        22h
ng-deploy-80   NodePort    10.100.34.200   <none>        80:30081/TCP   13s
## 访问service
root@k8s-master1:~/ceph-case# curl 172.31.2.1:30081/jike/
<h1>welcome to chaoge home</h1>
## 查看pod调度的节点
root@k8s-master1:~/ceph-case# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-796bf79987-7mtk5   1/1     Running   0          8m29s   10.200.169.148   k8s-node2   <none>           <none>
## 宿主机验证rbd
root@k8s-node2:~# rbd showmapped
id  pool           namespace  image         snap  device   
0   ygc-rbd-pool1             ygc-img-img1  -     /dev/rbd0
```
### 7.4.3 通过secret挂载rbd
```
## 将key进行base64编码
bash-4.4$ ceph auth print-key client.chaoge-ygc
AQAkJBNkFaQ1EBAAIGlG57NrXmlNh1ho+iKunA==
bash-4.4$ ceph auth print-key client.chaoge-ygc |base64
QVFBa0pCTmtGYVExRUJBQUlHbEc1N05yWG1sTmgxaG8raUt1bkE9PQ==
## secret yaml文件
root@k8s-master1:~/ceph-case# cat case3-secret-client-shijie.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-chaoge-ygc
type: "kubernetes.io/rbd"
data:
  key: QVFBa0pCTmtGYVExRUJBQUlHbEc1N05yWG1sTmgxaG8raUt1bkE9PQ== 
## 创建secret
root@k8s-master1:~/ceph-case# kubectl apply -f case3-secret-client-shijie.yaml 
secret/ceph-secret-chaoge-ygc created
## 查看secret
root@k8s-master1:~/ceph-case# kubectl get secret
NAME                     TYPE                DATA   AGE
ceph-secret-chaoge-ygc   kubernetes.io/rbd   1      7s
## deployment yaml 文件
root@k8s-master1:~/ceph-case# cat case4-nginx-secret.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - name: rbd-data1
          mountPath: /usr/share/nginx/html/rbd
      volumes:
        - name: rbd-data1
          rbd:
            monitors:
            - '10.100.135.254:6789'
            - '10.100.100.35:6789'
            - '10.100.232.104:6789'
            pool: ygc-rbd-pool1
            image: ygc-img-img1
            fsType: ext4
            readOnly: false
            user: chaoge-ygc
            secretRef:
              name: ceph-secret-chaoge-ygc
## 创建deployment
root@k8s-master1:~/ceph-case# kubectl apply -f case4-nginx-secret.yaml 
deployment.apps/nginx-deployment created
## 查看pod
root@k8s-master1:~/ceph-case# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-84b7b5c8c4-2f2q2   1/1     Running   0          24s   10.200.169.149   k8s-node2   <none>           <none>
## pod验证挂载
root@k8s-master1:~/ceph-case# kubectl exec -it nginx-deployment-84b7b5c8c4-2f2q2 -- bash
root@nginx-deployment-84b7b5c8c4-2f2q2:/# df -h
Filesystem                         Size  Used Avail Use% Mounted on
overlay                             97G   11G   82G  12% /
tmpfs                               64M     0   64M   0% /dev
/dev/mapper/ubuntu--vg-ubuntu--lv   97G   11G   82G  12% /etc/hosts
shm                                 64M     0   64M   0% /dev/shm
/dev/rbd0                          4.9G   28K  4.9G   1% /usr/share/nginx/html/rbd
tmpfs                              3.8G   12K  3.8G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                              1.9G     0  1.9G   0% /proc/acpi
tmpfs                              1.9G     0  1.9G   0% /proc/scsi
tmpfs                              1.9G     0  1.9G   0% /sys/firmware
## 宿主机验证挂
root@k8s-node2:~# rbd showmapped
id  pool           namespace  image         snap  device   
0   ygc-rbd-pool1             ygc-img-img1  -     /dev/rbd0
```
### 7.4.4 动态存储卷供给
```bash
## 进入rook csi实例rbd目录
root@k8s-master1:~# cd rook-1.11.1/deploy/examples/csi/rbd
## storageclass yaml文件
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/rbd# cat storageclass.yaml 
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph # namespace:cluster
spec:
  failureDomain: host
  replicated:
    size: 3
    # Disallow setting pool with replica 1, this could lead to data loss without recovery.
    # Make sure you're *ABSOLUTELY CERTAIN* that is what you want
    requireSafeReplicaSize: true
    # gives a hint (%) to Ceph in terms of expected consumption of the total cluster capacity of a given pool
    # for more info: https://docs.ceph.com/docs/master/rados/operations/placement-groups/#specifying-expected-pool-size
    #targetSizeRatio: .5
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph # namespace:cluster

  # If you want to use erasure coded pool with RBD, you need to create
  # two pools. one erasure coded and one replicated.
  # You need to specify the replicated pool here in the `pool` parameter, it is
  # used for the metadata of the images.
  # The erasure coded pool must be set as the `dataPool` parameter below.
  #dataPool: ec-data-pool
  pool: replicapool

  # (optional) mapOptions is a comma-separated list of map options.
  # For krbd options refer
  # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
  # For nbd options refer
  # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
  # mapOptions: lock_on_read,queue_depth=1024

  # (optional) unmapOptions is a comma-separated list of unmap options.
  # For krbd options refer
  # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
  # For nbd options refer
  # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
  # unmapOptions: force

   # (optional) Set it to true to encrypt each volume with encryption keys
   # from a key management system (KMS)
   # encrypted: "true"

   # (optional) Use external key management system (KMS) for encryption key by
   # specifying a unique ID matching a KMS ConfigMap. The ID is only used for
   # correlation to configmap entry.
   # encryptionKMSID: <kms-config-id>

  # RBD image format. Defaults to "2".
  imageFormat: "2"

  # RBD image features
  # Available for imageFormat: "2". Older releases of CSI RBD
  # support only the `layering` feature. The Linux kernel (KRBD) supports the
  # full complement of features as of 5.4
  # `layering` alone corresponds to Ceph's bitfield value of "2" ;
  # `layering` + `fast-diff` + `object-map` + `deep-flatten` + `exclusive-lock` together
  # correspond to Ceph's OR'd bitfield value of "63". Here we use
  # a symbolic, comma-separated format:
  # For 5.4 or later kernels:
  #imageFeatures: layering,fast-diff,object-map,deep-flatten,exclusive-lock
  # For 5.3 or earlier kernels:
  imageFeatures: layering

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
  # Specify the filesystem type of the volume. If not specified, csi-provisioner
  # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
  # in hyperconverged settings where the volume is mounted on the same node as the osds.
  csi.storage.k8s.io/fstype: ext4
# uncomment the following to use rbd-nbd as mounter on supported nodes
# **IMPORTANT**: CephCSI v3.4.0 onwards a volume healer functionality is added to reattach
# the PVC to application pod if nodeplugin pod restart.
# Its still in Alpha support. Therefore, this option is not recommended for production use.
#mounter: rbd-nbd
allowVolumeExpansion: true
reclaimPolicy: Delete
## 创建存储类
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/rbd# kubectl apply -f storageclass.yaml 
## 查看存储类
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/rbd# kubectl get storageclass
NAME              PROVISIONER                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-ceph-block   rook-ceph.rbd.csi.ceph.com      Delete          Immediate           true                   88s
rook-cephfs       rook-ceph.cephfs.csi.ceph.com   Delete          Immediate           true                   5h22m
rook-nfs          rook-ceph.nfs.csi.ceph.com      Delete          Immediate           true                   5h21m
## pvc yaml文件
root@k8s-master1:~/ceph-case# cat case7-mysql-pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: '5Gi'
## 创建pvc
root@k8s-master1:~/ceph-case# kubectl apply -f case7-mysql-pvc.yaml 
persistentvolumeclaim/mysql-data-pvc created
## 查看pvc,pv
root@k8s-master1:~/ceph-case# kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
mysql-data-pvc   Bound    pvc-5023b606-401f-42fb-a1e1-2d6f95585e08   5Gi        RWO            rook-ceph-block   2m20s
root@k8s-master1:~/ceph-case# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS      REASON   AGE
pvc-5023b606-401f-42fb-a1e1-2d6f95585e08   5Gi        RWO            Delete           Bound    default/mysql-data-pvc   rook-ceph-block            3m10s
## ceph 验证是否自动创建image
bash-4.4$ rbd ls --pool replicapool
csi-vol-8ff48152-1944-4950-a7eb-d0102ebaf713
## 单机mysql yaml 文
root@k8s-master1:~/ceph-case# cat case8-mysql-single.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6.46
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: magedu123456
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-data-pvc 


---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: mysql-service-label 
  name: mysql-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 3306
    protocol: TCP
    targetPort: 3306
    nodePort: 33306
  selector:
    app: mysql
root@k8s-master1:~/ceph-case# cat case8-mysql-single.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:5.6.46
        name: mysql
        env:
          # Use secret in real usage
        - name: MYSQL_ROOT_PASSWORD
          value: magedu123456
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-data-pvc 


---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: mysql-service-label 
  name: mysql-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 3306
    protocol: TCP
    targetPort: 3306
    nodePort: 30306
  selector:
    app: mysql
## 创建MySQLroot@k8s-master1:~/ceph-case# kubectl apply -f case8-mysql-single.yaml
## 查看pod和service
root@k8s-master1:~/ceph-case# kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-86b69d54b5-hvsfn   1/1     Running   0          2m25s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/kubernetes      ClusterIP   10.100.0.1      <none>        443/TCP          23h
service/mysql-service   NodePort    10.100.60.216   <none>        3306:30306/TCP   110s
## 验证mysql 挂载
root@k8s-master1:~/ceph-case# kubectl exec -it mysql-86b69d54b5-hvsfn -- bash
root@mysql-86b69d54b5-hvsfn:/# df -h
Filesystem                         Size  Used Avail Use% Mounted on
overlay                             97G   11G   81G  12% /
tmpfs                               64M     0   64M   0% /dev
/dev/mapper/ubuntu--vg-ubuntu--lv   97G   11G   81G  12% /etc/hosts
shm                                 64M     0   64M   0% /dev/shm
/dev/rbd1                          4.9G  116M  4.8G   3% /var/lib/mysql
tmpfs                              3.8G   12K  3.8G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                              1.9G     0  1.9G   0% /proc/acpi
tmpfs                              1.9G     0  1.9G   0% /proc/scsi
tmpfs                              1.9G     0  1.9G   0% /sys/firmware
## 验证mysql 访问
root@k8s-master1:~/ceph-case# apt install mysql-client -y
root@k8s-master1:~/ceph-case# mysql -h172.31.2.1 -P30306 -uroot -pmagedu123456 
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.46 MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```
### 7.4.5 cephfs 存储类使用
```bash
## cephfs存储类
root@k8s-master1:~# cd rook-1.11.1/deploy/examples/csi/cephfs
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/cephfs# cat storageclass.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com # driver:namespace:operator
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph # namespace:cluster

  # CephFS filesystem name into which the volume shall be created
  fsName: myfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: myfs-replicated

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster

  # (optional) The driver can use either ceph-fuse (fuse) or ceph kernel client (kernel)
  # If omitted, default volume mounter will be used - this is determined by probing for ceph-fuse
  # or by setting the default mounter explicitly via --volumemounter command-line argument.
  # mounter: kernel
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  # uncomment the following line for debugging
  #- debug
## 创建存储类
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/cephfs# kubectl apply -f storageclass.yaml 
## cephfs pvc yaml
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/cephfs# cat pvc.yaml 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: rook-cephfs
## 创建pvc
root@k8s-master1:~/rook-1.11.1/deploy/examples/csi/cephfs# kubectl get pvc,pv
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/cephfs-pvc   Bound    pvc-5fd0d29c-b72c-4f29-880c-602db501b269   10Gi       RWX            rook-cephfs    11s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS   REASON   AGE
persistentvolume/pvc-5fd0d29c-b72c-4f29-880c-602db501b269   10Gi       RWX            Delete           Bound    default/cephfs-pvc   rook-cephfs             11s

## deployment yaml文件
root@k8s-master1:~/ceph-case# cat case9-nginx-cephfs.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels: #rs or deployment
      app: ng-deploy-80
  template:
    metadata:
      labels:
        app: ng-deploy-80
    spec:
      containers:
      - name: ng-deploy-80
        image: nginx
        ports:
        - containerPort: 80

        volumeMounts:
        - name: magedu-staticdata-cephfs 
          mountPath: /usr/share/nginx/html/cephfs
      volumes:
        - name: magedu-staticdata-cephfs
          persistentVolumeClaim:
            claimName: cephfs-pvc

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: ng-deploy-80-service-label
  name: ng-deploy-80-service
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 32380
  selector:
    app: ng-deploy-80
## 创建deploym
root@k8s-master1:~/ceph-case# kubectl apply -f case9-nginx-cephfs.yaml
## 查看pod和svc
root@k8s-master1:~/ceph-case# kubectl get pod,svc
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-867d745c5d-sg4md   1/1     Running   0          46s
pod/nginx-deployment-867d745c5d-xppfm   1/1     Running   0          46s

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes             ClusterIP   10.100.0.1      <none>        443/TCP        24h
service/ng-deploy-80-service   NodePort    10.100.53.206   <none>        80:32380/TCP   46s
## pod 验证挂载
root@k8s-master1:~/ceph-case# kubectl exec -it nginx-deployment-867d745c5d-sg4md -- bash
root@nginx-deployment-867d745c5d-sg4md:/# df -h
Filesystem                                                                                                                                                 Size  Used Avail Use% Mounted on
overlay                                                                                                                                                     97G   11G   82G  12% /
tmpfs                                                                                                                                                       64M     0   64M   0% /dev
/dev/mapper/ubuntu--vg-ubuntu--lv                                                                                                                           97G   11G   82G  12% /etc/hosts
shm                                                                                                                                                         64M     0   64M   0% /dev/shm
tmpfs                                                                                                                                                      3.8G   12K  3.8G   1% /run/secrets/kubernetes.io/serviceaccount
10.100.135.254:6789,10.100.100.35:6789,10.100.232.104:6789:/volumes/csi/csi-vol-98258bc8-5d6e-474e-a148-0195b4885729/de1fcb83-9ef3-4a52-ac6d-946388a878d1   10G     0   10G   0% /usr/share/nginx/html/cephfs
tmpfs                                                                                                                                                      1.9G     0  1.9G   0% /proc/acpi
tmpfs                                                                                                                                                      1.9G     0  1.9G   0% /proc/scsi
tmpfs                                                                                                                                                      1.9G     0  1.9G   0% /sys/firmware
## pod 多副本验证
root@k8s-master1:~/ceph-case# vi case9-nginx-cephfs.yaml
replicas: 4
## 应用yaml文件
root@k8s-master1:~/ceph-case# kubectl apply -f case9-nginx-cephfs.yaml 
deployment.apps/nginx-deployment configured
service/ng-deploy-80-service unchanged
## 查看pod
root@k8s-master1:~/ceph-case# kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
nginx-deployment-867d745c5d-554ms   1/1     Running   0          70s     10.200.169.152   k8s-node2   <none>           <none>
nginx-deployment-867d745c5d-sg4md   1/1     Running   0          4m55s   10.200.36.83     k8s-node1   <none>           <none>
nginx-deployment-867d745c5d-vfdns   1/1     Running   0          70s     10.200.107.210   k8s-node3   <none>           <none>
nginx-deployment-867d745c5d-xppfm   1/1     Running   0          4m55s   10.200.169.151   k8s-node2   <none>           <none>

## 节点验证
root@k8s-node1:~# df -h | grep pvc
10.100.135.254:6789,10.100.100.35:6789,10.100.232.104:6789:/volumes/csi/csi-vol-98258bc8-5d6e-474e-a148-0195b4885729/de1fcb83-9ef3-4a52-ac6d-946388a878d1   10G     0   10G   0% /var/lib/kubelet/pods/f05470aa-6e11-4e7e-9fce-e06e1fb00d58/volumes/kubernetes.io~csi/pvc-5fd0d29c-b72c-4f29-880c-602db501b269/mount
## cephfs共享验证
root@k8s-master1:~/ceph-case# kubectl exec -it nginx-deployment-867d745c5d-sg4md -- bash
root@nginx-deployment-867d745c5d-sg4md:~# cd /usr/share/nginx/html/cephfs
## 共享目录写入index.html
root@nginx-deployment-867d745c5d-sg4md:/usr/share/nginx/html/cephfs# echo "<h1>hell ceph</h1>" >> index.html
## 访问每个pod的nginx
root@k8s-master1:~/ceph-case# curl 10.200.169.152/cephfs/
<h1>hell ceph</h1>
root@k8s-master1:~/ceph-case# curl 10.200.36.83/cephfs/
<h1>hell ceph</h1>
root@k8s-master1:~/ceph-case# curl 10.200.107.210/cephfs/
<h1>hell ceph</h1>
root@k8s-master1:~/ceph-case# curl 10.200.169.151/cephfs/
<h1>hell ceph</h1>
```