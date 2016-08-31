# 组件

apiserver: Kubernetes API server 为 api 对象验证及配置数据, 包括 pod, service, replicationcontroller 及其他. API Server 通过 REST 操作,对集群的共享状态提供前端,通过此,其他组件得以交互.

controller-manager: Kubernetes controller manager 是一个守护进程, 嵌入了与 Kubernetes 一起部署的核心控制回路. 在机械化和自动化的应用中, 控制回路是一种控制系统状态的非终结回路. 在 Kubernetes 中, controller 是用来通过 apiserver 监控集群的共享状态,并做出变更试图使得当前状态符合期望状态. 当前与 Kubernetes 一同部署的 controller 例子为 replication controller, endpoints controller, namespace
controller, 及 serviceaccounts controller.

scheduler: Kubernetes scheduler 是提供丰富策略, 拓扑感知, 特定工作负载的功能, 特别关注于可用性, 性能和容量. scheduler 需要考虑到账号的独立和集中的资源需求, 服务的质量需求, 硬件/软件/策略的约束, 亲和和反亲和规格, 数据位置, 内部工作负载干扰, 截止时间及其他.特定工作负载需求在需要时将会通过 API 暴露.

proxy: Kubernetes network proxy 在每个节点上运行. 此反射服务如在每个节点上 Kubernetes API 中的定义, 能通过一组后端执行简单 TCP,UDP 流转发或 round robin TCP,UDP 转发. 服务集群的 ip 和端口当前通过 Docker-links-compatible 环境变量指定端口查找, 由服务 proxy 开启. 此处有可选插件用于对这些集群 IP 提供集群 DNS. 用户必须通过 apiserver API 创建服务来配置 proxy.

kubelet: kubelet是主要的 "节点代理", 运行于每个节点上. kubelet 依据 PodSpec 工作. PodSpec 是一个描述了 pod 的 YAML 或 JSON 对象. kubelet 接受一组通过各种机制(主要通过 apiserver)提供的 PodSpec,并确保 PodSpec 中所描述容器的运行和健康.除了apiserver,还可以以文件,http endpoint, http server 方式接收 podspec.

# 概念

pod: 基本操作单元,封装一个或多个容器, 共享相同 namespace,因此在相同的volume, network上.

service: 基本操作单元, 应用服务抽象. service 后端由多个 pod 提供支持. 对外表现单一的访问接口.

replication controller: 根据 pod 模版启动 pod, 确保有副本数数量的 pod 在运行(rescheduling);根据调整的副本数调整 pod 运行(scaling);逐个替换的方式升级 pod (rolling update)


# 架构

etcd 作为数据存储位置, 被 apiserver 所使用

apiserver作为接口提供方, 被 controller-manager, scheduler, proxy, kubelet 所使用




