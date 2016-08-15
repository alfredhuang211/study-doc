# kube-deploy docker multinode部署方式分析

标签： kubernetes k8s kube-deploy 部署 容器

---

由于 k8s 使用容器方式部署的方便, 现对 k8s 的 kube-deploy 项目中的 docker 部署方式进行一定的分析.

## master 部署分析

入口为 master.sh, 实际内部仅调用了若干函数. 主要分支为根据 USE_CNI 环境变量的配置, 决定 CNI 启动于否. 

脚本首先设置 MASTER_IP 为 localhost.

调用 main 函数
> docker 检查
> 执行权限需要为root
> 确定K8S_VERSION为用户定义或最新
> ETCD_VERSION 确定
> FLANNEL_VERSION 及相关版本信息确定
> 系统架构: x86, arm...
> 使用的网口: eth0...
> 使用的IP地址
> bootstrap docker 的 sock 地址, 默认 unix:///var/run/docker-bootstrap.sock
> etcd的网络配置: --net host: 默认是用主机网络
> KUBELET_MOUNTS 挂载参数
> 如果使用CNI:
>> bootstrap 参数修改
>> etcd网络配置参数变更为: "-p 2379:2379 -p 2380:2380 -p 4001:4001"
>> CNI 参数配置

调用 log_variables 函数
> 展示各项参数

调用 install_network_utils 函数
> 根据操作系统(yum 或 apt-get)安装 net-tools 和 bridge-utils 组件.

调用 turndown 函数
> 检查是否有 bootstrap docker, 如有, 清理里面的容器, 停止相关进程.
> 检查是否有 kube_ 或 k8s_ 相关容器, 如有, 清理容器.
> 是否清理/var/lib/kubelet, 如清理,卸载可能有的挂载, 删除目录
> 如果有 cni0 的 bridge 网口, 停止网口并删除

分支1:使用CNI

* 使用CNI, 调用cni的 ensure_docker_settings 函数
> 确保有 systemctl 命令
> 获取 docker 启动配置文件
> 清理里面的 mtu 和 bip 参数
> 通过增加 shared-mounts.conf 文件设置 docker 配置的 MountFlags=shared
> 根据需要重启 docker

* 使用CNI, 调用 start_etcd 函数
> 在当前 docker 启动 kube-etcd, 通过映射暴露服务端口,挂载 /var/lib/kubelet/etcd 目录作为数据目录
> 循环等待检查服务是否启动完成, 如果超时报错退出
> 使用镜像中的 etcdctl 设置 flannel 网络配置

* 使用CNI, 调用 start_flannel 函数
> 在当前 docker 启动 kube-flannel, net 为 host, privileged, 映射 /dev/net, FLANNEL_SUBNET_DIR, 
> 循环等待检查服务是否启动完成
> 读取 flannel 子网配置并显示

分支2:不使用CNI

* 不使用CNI, 调用 bootstrap_daemon 函数

> 启动 bootstrap docker, iptables为false, ip-masp为false, 无bridge, 
> 循环等待检查服务是否启动完成

* 不使用CNI, 调用 start_etcd 函数
> 相同, 不过由于配置了 bootstrap docker 参数, 容器会在 bootstrap docker中启动

* 不使用CNI, 调用 start_flannel 函数
> 相同, 不过由于配置了 bootstrap docker 参数, 容器会在 bootstrap docker中启动

* 不使用CNI, 调用 restart_docker 函数
> 如果有 systemctl 命令, 备份 docker 启动配置文件, 替换 mtu, bip 为 flannel 相关配置, 停止并删除 docker0 的bridge网口, 重启 docker 服务, 此 docker 为主 docker daemon.
> 如果有 yum 命令, 备份 /etc/sysconfig/docker, 替换 mtu, bip 为 flannel 相关配置,停止并删除 docker0 的bridge网口, 重启 docker 服务.
> 如果有 apt-get 命令, 备份 /etc/default/docker, 替换 mtu, bip 为 flannel 相关配置,停止并删除 docker0 的bridge网口, 重启 docker 服务.

分支完结,启动kubelet

调用 start_k8s_master 函数
> 准备 shared_kubelet_dir: 创建/var/lib/kubelet, mount --bind, mount --make-shared
> 启动 kube_kubelet 容器, 镜像为hyperkube, net使用host, pid使用host, privileged, 命令为/hyperkube kubelet, 参数包括 allow-privileged, api-server设置为http://localhost:8080, cluster-dns 和 cluster-domain 设置, hostname-override 为当前ip, config 为 /etc/kubernetes/manifests-multi

至此, master.sh 脚本执行完成, master 节点启动完成.

## worker 部署分析

入口为 worker.sh, 实际内部仅调用了若干函数. 主要分支为根据 USE_CNI 环境变量的配置, 决定 CNI 启动于否. 

在函数运行前, 检查 MASTER_IP 是否设置.

函数执行与 master.sh 相同, 仅在执行完 CNI 分支后, 执行的时 start_k8s_worker 和 start_k8s_worker_proxy 函数

调用 start_k8s_worker 函数
> 准备 shared_kubelet_dir: 创建/var/lib/kubelet, mount --bind, mount --make-shared
> 启动 kube_kubelet 容器, 镜像为hyperkube, net使用host, pid使用host, privileged, 命令为/hyperkube kubelet, 参数包括 allow-privileged, api-server设置为http://${MASTER_IP}:8080, cluster-dns 和 cluster-domain 设置, hostname-override 为当前ip.无config.

调用 start_k8s_worker_proxy 函数
> 启动 kube_proxy 容器, 镜像为hyperkube, net使用host, privileged, 命令为hyperkube proxy, 参数包括 --master=http://${MASTER_IP}:8080

至此, worker.sh 脚本执行完成, worker 节点启动完成.






