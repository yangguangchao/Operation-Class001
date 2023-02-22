# 1. 添加两个以上静态令牌认证的用户，例如tom和jerry，并认证到Kubernetes上；添加两个以上的X509证书认证到Kubernetes的用户，比如mason和magedu；把认证凭据添加到kubeconfig配置文件进行加载；
# 2. 使用资源配置文件创建ServiceAccount，并附加一个imagePullSecrets；
# 3. 为tom用户授予管理blog名称空间的权限；为jerry授予管理整个集群的权限；为mason用户授予读取集群资源的权限；
# 4. 部署Jenkins、Prometheus-Server、Node-Exporter至Kubernetes集群；而后使用Ingress开放至集群外部，Jenkins要使用https协议开放；
# 5. 使用helm部署主从复制的MySQL集群，部署wordpress，并使用ingress暴露到集群外部；使用helm部署harbor，成功验证推送Image至Harbor上；使用helm部署一个redis cluster至Kubernetes上；