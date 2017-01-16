# kubeadm代码分析


## kubeadm init 分析

1. 命令行处理,收取各种参数

2. Run函数执行
    
CreateTokenAuthFile : 通过设置的 token 或随机生成 token ,在 /etc/kubernetes/pki 下创建 token.csv 文件,文件内容例如 `9d8e3b90523c5e3f,kubeadm-node-csr,f387963f-d31b-11e6-bbc3-080027b2ff68,system:kubelet-bootstrap`
    
WriteStaticPodManifests : 通过pod配置,生成apiserver,controller manager,scheduler, etcd的配置json文件,放置在 `/etc/kubernetes/manifests` 下.

> 各配置文件分析:

> etcd.json: Pod类型, 包括三个 hostpath, `/etc/ssl/certs`, `/var/lib/etcd`, `/etc/kubernetes`, 使用 `etcd-amd64:3.0.14-kubeadm` 镜像启动容器, 挂载路径分别为 `/etc/ssl/certs`, `/var/etcd`, `/etc/kubernetes/`, 启动命令 `etcd --listen-client-urls=http://127.0.0.1:2379 --advertise-client-urls=http://127.0.0.1:2379 --data-dir=/var/etcd/data`, 网络使用 host 模式, 带有 cpu requests, livenessprobe, securityContext.

> kube-apiserver.json: Pod类型, 包括 `/etc/ssl/certs`, `/etc/kubernetes` 两个 hostpath, 使用 `kube-apiserver-amd64:v1.5.1` 镜像启动容器, 挂载路径分别为 `/etc/ssl/certs`, `/etc/kubernetes/`, 启动命令中较特殊的为 `service-account-key-file client-ca-file tls-cert-file tls-private-key-file token-auth-file kubelet-preferred-address-types anonymous-auth`, 网络使用 host 模式, 带有 cpu requests, livenessprobe.

> kube-controller-manager.json: Pod类型, 同api-server一样的hostpath和挂载路径, 使用 `kube-controller-manager-amd64:v1.5.1` 镜像启动容器, 启动命令中的特殊处为 `leader-elect cluster-name root-ca-file service-account-private-key-file cluster-signing-cert-file cluster-signing-key-file insecure-experimental-approve-all-kubelet-csrs-for-group allocate-node-cidrs cluster-cidr `, 网络使用 host 模式, 带有 cpu requests, livenessprobe.

> kube-scheduler.json: Pod类型, 使用 `kube-scheduler-amd64:v1.5.1` 镜像启动容器, 启动命令中带有 `leader-elect`, 网络使用 host 模式, 带有 cpu requests, livenessprobe.
    
CreatePKIAssets: 收集整理对外ip地址, dns地址, 生成rsa的私钥 cakey, 使用key生成自签名的cacert, 使用 `ca-pub.pem`, `ca-key.pem`, ca.pem 作为公钥,私钥和证书的文件名, 将 privatekey, 对应的publickey, cacert 分别写入对应文件. 使用集群内dns名,内部cidr相关的ip, 使用 cakey 和 cacert, 生成新的 私钥 key 和对应的 cert. 使用新生成的 key 和 cert, 使用 `apiserver-pub.pem`, `apiserver-key.pem`, `apiserver.pem` 作为公钥,私钥和证书的文件名,将其存入.生成新的rsa私钥,作为sa私钥,生成对应的公钥,并写入 `sa-key.pem` 和 `sa-pub.pem`. cakey和cacert返回.所有相关私钥,公钥和证书的放置位置均为 `/etc/kubernetes/pki`

CreateCertsAndConfigForClients: 使用"kubernetes"为集群名, 默认 https 地址为服务器名, cacert作为证书生成基础客户端配置, 通过cacert和cakey生成新的key和cert, 使用新生成的key和cert,配合基础客户端,生成kubelet和admin用的客户端配置

WriteKubeconfigIfNotExists: 将上步生成的客户端配置, 写入对应名称的 conf 文件, 主要就是 `admin.conf` 和 `kubelet.conf`, 放置位置为 `/etc/kubernetes`, 目前两文件内容相同, 内部均保存了 admin 配置和 kubelet 配置.

CreateClientAndWaitForAPI: 使用 admin 配置,创建 client 配置, 并用配置创建 client, 循环等待 client 完成连接, 通过 client 检查集群组件情况, 再次循环等待, 通过 client 获取到至少一个 node 完成注册. 创建,获取,删除 dummy deployment 用于检查集群已准备好. 返回 client.

UpdateMasterRoleLabelsAndTaints: 通过 client 更新 node 节点元数据 Taints ,标记为`dedicated=master:NoSchedule`, 确保普通 pod 调度不会到 master 上.

CreateDiscoveryDeploymentAndSecret: 通过 cacert 生成新的相关证书和配置的 json 文件,作为 secret, 保存为 clusterinfo 的名称. 创建名称为 kube-discovery 的 deployment, 启动 pod 并挂载使用 secret. 循环检查确保 deployment 完成启动.

CreateEssentialAddons: 创建 deamonset 类型的 kubeproxy 并启动, 创建 deployment 类型的 kube-dns 并启动, 创建 kube-dns 的服务.

generateJoinArgs: 使用 joinArgsTemplateLiteral 内容作为模版,根据各项配置,生成 node 上需要执行的 join 字段并输出.


## kubeadm join 分析

1. 命令行处理,收取各种参数

2. Run函数执行

RetrieveTrustedClusterInfo: 使用 master 地址, discovery 端口, token 前部, 生成相应的 url, 通过 http get 获取请求响应的数据, 通过 token 后部解密, 获取集群的相关配置信息. 

EstablishMasterConnection: 通过集群信息中的 endpoints, cacert, 创建client, 使用多个线程分别连接 endpoints 中的不通 endpoint. 通过获取 server 版本, certificate api 获取内容, 完成对 endpoint 的检查和可用判断.返回已经建立好的连接.

PerformTLSBootstrap: 使用已建立的连接创建认证请求 client, 创建 ecdsa 私钥, 通过 client, 请求节点认证信息, 从 api-server 获取 cert, 通过连接的 endpoint, cacert 生成基础信息, 通过基础信息, 自身节点名, 私钥和 cert, 生成最终的节点配置. 返回最终配置.

WriteKubeconfigIfNotExists: 将最终配置写入配置文件, 默认文件名为 `kubelet.conf`, 默认位置为 `/etc/kubernetes`.

