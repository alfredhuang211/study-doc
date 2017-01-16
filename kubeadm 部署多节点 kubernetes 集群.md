# kubeadm 进行多节点kubernetes 集群搭建

标签： kubernetes k8s 容器 安装部署 kubeadm

---
## 部署环境

系统环境: ubuntu 16.04, docker 1.12

设备准备: 三台设备, 分别为192.168.4.250/251/252, 其中 250 部署为 master, 另外两台部署为 node. 三台主机需求修改为不同 hostname 避免冲突.

环境准备: 修改hosts, 确保能够正常从 gcr.io 上完成镜像下载.

## 初始准备

在每台设备上均完成 kubelet, kubernetes-cni, kubeadm, kubectl 的安装.

1. 增加 apt-key 证书:
```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```
2. 增加相关源到apt中:
```bash
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
3. 更新源并安装相关组件:
```bash
apt-get update
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

## master 部署

由于计划使用 flannel 网络, 所以 kubeadm 的 init 命令需要带有参数 pod-network-cidr.

通过执行如下命令完成 master 的启动:
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
```

命令会自行完成主机依赖检查, 使用的镜像下载和 pod 启动.

最终输出中会有 `kubeadm join --token=4fd077.9d8e3b90523c5e3f 192.168.3.250` 类似字段, 用于在 node 上执行以便加入主机到集群中.

## flannel 网络安装

由于 flannel 现在也和 cni 进行了对接, 因此可以直接以 daemonset 的方式启动. 通过执行如下命令完成 flannel 网络组件的安装和启动.
```bash
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

此方式将自动使用 `https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml` 此位置保存的 yaml 文件来完成 configmap 和 daemonset 的创建.

## node 部署

利用 master 部署最终生成的命令,在 node 上执行:
```bash
kubeadm join --token=4fd077.9d8e3b90523c5e3f 192.168.3.250
```

命令同样会自行完成主机依赖检查, 相关配置生成, 使得主机上的 kubelet 能正确加入到 master 所管理的集群中去.

同时由于 flannel 和 kube-proxy 都是以 daemonset 的形式启动, 会在 node 加入集群后自动完成节点上 flannel 和 kube-proxy 的启动.

## 检查集群状态

最终通过 master 上 kubectl 相关命令完成集群情况的检查:
```bash
# kubectl version
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.1", GitCommit:"82450d03cb057bab0950214ef122b67c83fb11df", GitTreeState:"clean", BuildDate:"2016-12-14T00:57:05Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.1", GitCommit:"82450d03cb057bab0950214ef122b67c83fb11df", GitTreeState:"clean", BuildDate:"2016-12-14T00:52:01Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}


# kubectl get node
NAME       STATUS         AGE
1604       Ready,master   1d
1604.251   Ready          1d
1604.252   Ready          1d

kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
```


