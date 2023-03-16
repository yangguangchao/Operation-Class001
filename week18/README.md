# 1. 在 K8s 环境基于 daemonset 部署日志收集组件实现 pod 日志收集
# 2. 在 K8s 环境对 pod 添加 sidecar 容器实现业务日志收集
# 3. 在 K8s 环境容器中启动日志收集进程实现业务日志收集
# 4. 通过 prometheus 对 CoreDNS 进行监控并在 grafana 显示监控图形
# 5. 对 K8s 集群进行 master 节点扩容、node 节点扩容
# 6. 对 K8s 集群进行小版本升级
# 7. 基于 ceph rbd 及 cephfs 持久化 K8s 中 pod 的业务数据