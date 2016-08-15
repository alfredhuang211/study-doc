# kubernetes 容器方式部署

标签： kubernetes k8s 容器 安装部署

---
## 部署环境

系统环境: ubuntu 16.04, docker 1.12

设备准备: 两台设备, 分别为192.168.4.220/219, 其中 220 部署为 master, 219 部署为 worker.

部署使用: 使用 [kube-deploy](https://github.com/kubernetes/kube-deploy) 项目完成部署

注意事项: 由于部署过程中会去 gcr.io 上下载镜像, 国内环境可能导致下载失败.可以将 `220.255.2.153 gcr.io ` `220.255.2.153 storage.googleapis.com` 添加到 hosts 文件中确保解析能够正确, 镜像能够下载.

## 部署操作

### master 部署

在master主机上执行如下命令:
```bash
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ ./master.sh
```

输出如下:
```bash
+++ [0812 14:59:36] K8S_VERSION is set to: v1.3.5
+++ [0812 14:59:36] ETCD_VERSION is set to: 2.2.5
+++ [0812 14:59:36] FLANNEL_VERSION is set to: 0.5.5
+++ [0812 14:59:36] FLANNEL_IPMASQ is set to: true
+++ [0812 14:59:36] FLANNEL_NETWORK is set to: 10.1.0.0/16
+++ [0812 14:59:36] FLANNEL_BACKEND is set to: udp
+++ [0812 14:59:36] RESTART_POLICY is set to: unless-stopped
+++ [0812 14:59:36] MASTER_IP is set to: localhost
+++ [0812 14:59:36] ARCH is set to: amd64
+++ [0812 14:59:36] IP_ADDRESS is set to: 192.168.4.220
+++ [0812 14:59:36] USE_CNI is set to: false
+++ [0812 14:59:36] --------------------------------------------
+++ [0812 14:59:36] Killing all kubernetes containers...
+++ [0812 14:59:36] Launching docker bootstrap...
+++ [0812 14:59:37] Launching etcd...
+++ [0812 15:01:46] Launching flannel...
{"action":"set","node":{"key":"/coreos.com/network/config","value":"{ \"Network\": \"10.1.0.0/16\", \"Backend\": {\"Type\": \"udp\"}}","modifiedIndex":4,"createdIndex":4}}
+++ [0812 15:05:34] FLANNEL_SUBNET is set to: 10.1.84.1/24
+++ [0812 15:05:34] FLANNEL_MTU is set to: 1472
+++ [0812 15:05:34] Restarting main docker daemon...
+++ [0812 15:05:36] Restarted docker with the new flannel settings
+++ [0812 15:05:36] Launching Kubernetes master components...
+++ [0812 15:16:49] Done. It may take about a minute before apiserver is up.
```

在此过程中会下载如下镜像,如果不能自动拉取,在安装前安装好镜像应该也是可以的.
```bash
gcr.io/google_containers/hyperkube-amd64
gcr.io/google_containers/pause-amd64
gcr.io/google_containers/etcd-amd64
gcr.io/google_containers/flannel-amd64
```

### worker 部署

在 worker 主机上执行如下命令:
```bash
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ export MASTER_IP=192.168.4.220
$ ./worker.sh
```

输出如下:
```bash
+++ [0812 15:50:51] K8S_VERSION is set to: v1.3.5
+++ [0812 15:50:51] ETCD_VERSION is set to: 2.2.5
+++ [0812 15:50:51] FLANNEL_VERSION is set to: 0.5.5
+++ [0812 15:50:51] FLANNEL_IPMASQ is set to: true
+++ [0812 15:50:51] FLANNEL_NETWORK is set to: 10.1.0.0/16
+++ [0812 15:50:51] FLANNEL_BACKEND is set to: udp
+++ [0812 15:50:51] RESTART_POLICY is set to: unless-stopped
+++ [0812 15:50:51] MASTER_IP is set to: 192.168.4.220
+++ [0812 15:50:51] ARCH is set to: amd64
+++ [0812 15:50:51] IP_ADDRESS is set to: 192.168.4.219
+++ [0812 15:50:51] USE_CNI is set to: false
+++ [0812 15:50:51] --------------------------------------------
+++ [0812 15:50:51] Killing all kubernetes containers...
+++ [0812 15:50:51] Launching docker bootstrap...
+++ [0812 15:50:52] Launching flannel...
+++ [0812 15:54:08] FLANNEL_SUBNET is set to: 10.1.13.1/24
+++ [0812 15:54:08] FLANNEL_MTU is set to: 1472
+++ [0812 15:54:08] Restarting main docker daemon...
+++ [0812 15:54:10] Restarted docker with the new flannel settings
+++ [0812 15:54:10] Launching Kubernetes worker components...
+++ [0812 16:01:24] Launching kube-proxy...
+++ [0812 16:01:24] Done. After about a minute the node should be ready.
```

在此过程中会下载如下镜像,如果不能自动拉取,在安装前安装好镜像应该也是可以的.
```bash
gcr.io/google_containers/hyperkube-amd64
gcr.io/google_containers/flannel-amd64
```

### dashboard访问

由于部署了 k8s 的 dashboard, 因此可以直接访问 dashboard 查看到页面:
`http://192.168.4.220:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/workload` 

### kubectl 下载安装

从 kubernetes 发行 release 中可以找到对应的 kubectl 可执行文件.将其放置到 master 中.

执行 `kubectl version`
输出如下:
```bash
root@1604:~/kube-deploy/docker-multinode# kubectl version
Client Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.4", GitCommit:"dd6b458ef8dbf24aff55795baa68f83383c9b3a9", GitTreeState:"clean", BuildDate:"2016-08-01T16:45:16Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.5", GitCommit:"b0deb2eb8f4037421077f77cb163dbb4c0a2a9f5", GitTreeState:"clean", BuildDate:"2016-08-11T20:21:58Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
```
可以查看到正确的 server 信息.


