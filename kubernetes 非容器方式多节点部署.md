# kubernetes 非容器方式多节点部署

标签： kubernetes k8s 部署 ubuntu docker

---

## 部署环境:
操作系统: ubuntu 14.04.4 (ubuntu 15+ 由于 systemd 的使用, 在20160813日期时还未支持)
docker: 使用的是 1.11.2 版本, 官方文档指明仅需1.2+即可.
bridge-utils: 使用 apt-get install bridge-utils 安装.
hosts: 将 `220.255.2.153 storage.googleapis.com` 添加到 `/etc/hosts` 避免国内环境导致的下载失败.

实际操作主机:
192.168.4.218 192.168.4.217 两台主机, 其中 218 部署为 master+worker, 217 部署为 worker.

## 部署步骤
### 代码库下载
> git clone --depth 1 https://github.com/kubernetes/kubernetes.git
    
### 集群配置

1. 确定etcd, flannel, kubernetes的版本, 如果均不输入, kubernetes 默认为最新 release, etcd 默认为 2.3.1, flannel 默认为 0.5.5
> export KUBE_VERSION=1.2.0
> export FLANNEL_VERSION=0.5.0
> export ETCD_VERSION=2.2.0

2. 确定集群配置
> export nodes="root@192.168.4.218 root@192.168.4.217"
> export role="ai i"
> export NUM_NODES=${NUM_NODES:-2}
> export SERVICE_CLUSTER_IP_RANGE=10.0.1.0/24
> export FLANNEL_NET=172.16.0.0/16
> 其中 role中 ai 表示部署 master 和 worker, i 表示部署 worker.
> 如果节点更多, 可以根据实际情况修改 nodes 和 role.

3. 确定使用 ubuntu 为 KUBERNETES_PROVIDER
> export KUBERNETES_PROVIDER=ubuntu
> 此处如果非 ubuntu 系统,可以替换为实际使用的系统.

4. 执行部署
> ./kube-up.sh

### 集群验证
执行如下命令确认 kubernetes 已经正常启动
```bash
kubectl version
Client Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.4", GitCommit:"dd6b458ef8dbf24aff55795baa68f83383c9b3a9", GitTreeState:"clean", BuildDate:"2016-08-01T16:45:16Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"3", GitVersion:"v1.3.4", GitCommit:"dd6b458ef8dbf24aff55795baa68f83383c9b3a9", GitTreeState:"clean", BuildDate:"2016-08-01T16:38:31Z", GoVersion:"go1.6.2", Compiler:"gc", Platform:"linux/amd64"}
```

### dashboard 安装
通过如下命令可以完成 dashboard 的安装:
`kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml`

界面输出为:
```bash
deployment "kubernetes-dashboard" created
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:32628) to serve traffic.

See http://releases.k8s.io/release-1.3/docs/user-guide/services-firewalls.md for more details.
service "kubernetes-dashboard" created
```
此时可以通过 kubectl 命令查看部署情况:
```bash
root@ubuntu:~# kubectl get deployments --namespace=kube-system
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1         1         1            1           7m
root@ubuntu:~# kubectl get po --namespace=kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
kubernetes-dashboard-3825951078-c2w0i   1/1       Running   0          7m
root@ubuntu:~# kubectl get rs --namespace=kube-system
NAME                              DESIRED   CURRENT   AGE
kubernetes-dashboard-3825951078   1         1         8m
```

dashboard 默认部署到 namespace 为 kube-system 下.

此时可以通过 `http://{IP}:8080/ui` 访问dashboard, 链接会默认跳转到 `http://192.168.4.218:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/workload`, 如果 dashboard 未安装或安装失败, 此页面会报响应错误.


### 说明
在 kube-up.sh 执行过程中, 会有多次要求输入 master 主机和 worker 主机密码的过程. 可以通过配置 ssh 免密码登录来避免多次输入密码.
> 在主机上使用 ssh-keygen 生成 public-key, 默认位置为 ~/.ssh/id_rsa.pub
> 将其中内容复制到 master 主机上的 ~/.ssh/authorized_keys 内

在 kube-up.sh 执行过程中, 会根据模块版本号, 下载 etcd, flanner, kubernetes 的 release 包.在网速较慢或已经自行下载了模块版本的情况下,可以通过修改 cluser/ubuntu/download-release.sh, 避免重复下载.




