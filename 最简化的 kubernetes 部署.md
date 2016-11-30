# 最简化的 kubernetes 部署

标签：kubernetes 部署 k8s hyperkube flannel

---

本文说明最简单的 kubernetes 部署过程, 部署使用二进制文件, 从中可以了解 kubernetes 各组件的基本工作方式和组件协同方式.

## 部署工具
二进制文件: hyperkube, etcd, etcdctl, flanneld, kubectl

各文件说明:

`etcd`: 启动 etcd 服务或服务集群.

`etcdctl`: 检查 etcd 启动情况, 在 etcd 中读写一些值.

`flanneld`:  启动 flannel 网络.

`hyperkube`:  启动 kubernetes 各组件.

`kubectl`: 进行 kubernetes 配置操作.

## 部署环境

演示环境使用 ubunut 16.04. 由于二进制文件本身为 x64 版本,实际可以考虑部署并不依赖操作系统.

## etcd 启动

执行命令如下:
```
nohup ./etcd -name kube -listen-client-urls http://0.0.0.0:2379 -advertise-client-urls http://192.168.4.240:2379 > etcd.out &
```

## flannel 配置及启动

执行如下命令在 etcd 中设置好 flannel 的相关运行参数:
```
etcdctl mk /coreos.com/network/config '{ "Network": "172.17.0.0/16","Backend":{"Type":"vxlan"} }'
```

其中 `172.17.0.0/16` 指明了 flannel 可以使用的网段

执行如下命令启动 flannel :
```
nohup ./flanneld --etcd-endpoints=http://192.168.4.240:2379  --ip-masq  --iface=192.168.4.240 > flannel.out &
```

其中 etcd-endpoints 参数要求指向 etcd 的服务位置, iface 使用主机ip地址.

此时使用 `ifconfig` 可以看到多了 flannel 的网络设备, 通过 `cat /run/flannel/subnet.env` 可以看到本机 flannel 分配的相关信息,其中 FLANNEL_SUBNET 需要用来设置为 docker 的 bip, mtu 可设置为 docker 的 mtu.
FLANNEL_NETWORK=172.16.0.0/16
FLANNEL_SUBNET=172.16.46.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true


通过 `ifconfig docker0 172.17.41.1/24` 命令, 将 FLANNEL_SUBNET 的值设置给 docker0.

flannel 启动及 docker0 配置需要在每台 minion主机上都执行,确保接入集群运行 pod 的 docker 都正确配置了 flannel 网络.

## master 节点服务启动

apiserver 启动:
```
nohup ./hyperkube apiserver \
--insecure-bind-address=0.0.0.0 --insecure-port=8080 \
--etcd-servers=http://192.168.4.240:2379 --logtostderr=true \
--admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ResourceQuota \
--service-node-port-range=30000-32767 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.1.0/24 \
--advertise-address=192.168.4.240 \
> apiserver.out &
```

controller-manager 启动:
```
nohup ./hyperkube controller-manager \
--master=192.168.4.240:8080 \
--logtostderr=true \
> controller.out &
```

scheduler 启动:
```
nohup ./hyperkube scheduler \
--master=192.168.4.240:8080 \
--logtostderr=true \
> scheduler.out &
```

其中 etcd 地址, master 地址均根据实际情况进行填写.

此时可以通过 kubectl 查看相关信息:

通过 `./kubectl version` 查看拉起的 server 版本信息

通过 `./kubectl get cs` 查看组件健康状态


## minion 节点服务启动

进行节点 flannel 的启动和配置.

启动 proxy:
```
nohup ./hyperkube proxy \
--hostname-override=192.168.4.215 \
--master=http://192.168.4.240:8080 \
--logtostderr=true \
> proxy.out &
```
启动 kubelet:
```
nohup ./hyperkube kubelet \
--hostname-override=192.168.4.215 \
--api-servers=http://192.168.4.240:8080 \
--logtostderr=true \
--cluster-dns=192.168.3.10  --cluster-domain=cluster.local \
--allow-privileged=true \
> kubelet.out &
```

命令中的 hostname-override 可以自行起名, 或用当前主机 ip 地址替代. 

命令中的 master 和 api-server 都是使用 master 服务地址.

## 检查及验证

在 master 上使用 `kubectl` 检查节点情况:

通过`./kubectl get no` 能够查看节点的准备情况.

在 master 上使用命令启动 nginx :

通过命令 `./kubectl run my-nginx --image=nginx --replicas=2 --port=80` 在 kubernetes 中启动 nginx 服务.

启动后可通过 `./kubectl get po` 查看 pod 的启动情况.

由于在启动 pod 时, 集群会拉取 pause 镜像,可以通过 `echo "220.255.2.153 www.gcr.io" >> /etc/hosts` `echo "220.255.2.153 gcr.io" >> /etc/hosts` 来确保拉取镜像能够成功.

## 注意

由于这种方式部署,未考虑 kubernetes 的认证或 service account, 在拉起 kubernetes dashboard 的时候, 会使得 pod 处于 CrashLoopBackOff 的状态. docker logs 查看 dashboard 容器, 可以看到报错内容内有 "it has invalid apiserver certificates or service accounts configuration"

后续将进一步分析 apiserver certificates 和 service accounts 配置.



