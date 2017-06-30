# calico 安装及配置实现docker跨主机网络

标签： calico docker 跨主机 网络

---
## 说明：

本文通过配置使用 calico 最新版本，实现 docker 容器的跨主机通讯。

Calico 是一个纯3层协议，支持VM、Docker、Rocket、OpenStack、Kubernetes、或者直接在物理机上使用。官网上给出可以支持上万个主机、上百万的工作负载（container），由于它是纯三层协议，使用BGP协议（基于IP），更易于调试，支持IPv6，支持灵活的安全策略。

## 环境：

ubuntu 14.04.4 两台，安装的是docker 1.11.1 版本，ip 地址分别为 `192.168.3.229`，`192.168.3.228`

etcd 2.3.6

calicoctl v1.0.0-beta

## 准备:

在每台主机上都安装或下载好 calicoctl, 可通过命令 `wget https://github.com/projectcalico/calico-containers/releases/download/v1.0.0-beta/calicoctl` 完成下载。

启动 etcd，可使用如下命令手工启动 etcd 并运行到后台: `nohup ./etcd -name kube -listen-client-urls http://0.0.0.0:2379 -advertise-client-urls http://192.168.4.229:2379 > etcd.out &`

分别在两台主机上都配置 docker 使用 etcd 作为 cluster-store，在 `/etc/docker/daemon.json` 内配置或新增如下内容：
```json
{
"cluster-store":"etcd://192.168.3.229:2379"
}
```
配置完成后重启 docker 服务以便配置生效。

在每台主机上可以预先拉取 `calico/node:v1.0.0-beta` 镜像。

## 启动calico服务:

在每台主机上均运行命令：
```bash
export ETCD_ENDPOINTS=http://192.168.3.229:2379
./calicoctl node run --name 192.168.3.229
```
其中 calicoctl 命令里的 name，每台主机均可以根据自身 ip 来填写。
命令实际使用 calico/node 镜像启动了一个容器，执行输出内容如下：
```bash
Running command to load modules: modprobe -a xt_set ip6_tables
Enabling IPv4 forwarding
Enabling IPv6 forwarding
Increasing contrack limit
Running the following command:

docker run -d --net=host --privileged --name=calico-node -e IP6= -e NO_DEFAULT_POOLS= -e ETCD_SCHEME= -e CALICO_LIBNETWORK_ENABLED=true -e HOSTNAME=192.168.3.229 -e AS= -e ETCD_ENDPOINTS=http://192.168.3.229:2379 -e ETCD_AUTHORITY= -e IP= -e CALICO_NETWORKING_BACKEND=bird -v /var/run/calico:/var/run/calico -v /lib/modules:/lib/modules -v /run/docker/plugins:/run/docker/plugins -v /var/run/docker.sock:/var/run/docker.sock -v /var/log/calico:/var/log/calico calico/node:v1.0.0-beta
```

每台主机均启动 node 后，可查看 calico 内的 node 情况：
```bash
./calicoctl get node
NAME            
192.168.3.228   
192.168.3.229   

./calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.3.228 | node-to-node mesh | up    | 07:21:09 | Established |
+---------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

```

## 配置网络 IP Pool 
通过如下命令可以查看 calico 的默认 ip pool
```bash
 ./calicoctl get ipPool
CIDR                       
192.168.0.0/16             
fd80:24e2:f998:72d6::/64   
```
可以看到其默认的 ipv4 的 ip 池为 192.168.0.0/16
我们通过如下 yaml 文件创建新的 ip 池：
```yaml
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 10.10.0.0/16
  spec:
    ipip:
      enabled: true
    nat-outgoing: true
```
执行命令 `./calicoctl create -f calico.yaml` 创建此 ip pool，然后通过 `./calicoctl delete ipPool 192.168.0.0/16` 删除原来的 ip pool。

## 网络创建
由于 calico 已经对接了 docker 的 CNM 网络模型，因此网络的创建，删除等管理动作，都可以在 docker 内完成。
通过如下命令创建和查看 calico 网络：
```bash
docker network create --driver calico --ipam-driver calico-ipam net1

docker network create --driver calico --ipam-driver calico-ipam net2

docker network ls
NETWORK ID          NAME                DRIVER
c11b915fd760        bridge              bridge              
9f5a5025c65a        host                host                
8787da6413ac        net1                calico              
32cd38e0980b        net2                calico              
20a70c2f57c1        none                null     
```

## 跨主机通讯验证
在第一个节点上创建如下容器：
```bash
docker run --net net1 --name workload-A -tid busybox

docker run --net net2 --name workload-B -tid busybox

docker run --net net1 --name workload-C -tid busybox
```
在第二个节点上创建如下容器：
```bash
docker run --net net2 --name workload-D -tid busybox

docker run --net net1 --name workload-E -tid busybox
```

通过 ping 命令查看网络连通性：
```bash
docker exec workload-A ping -c 4 workload-C //同主机通讯同网络通讯
PING workload-C (10.10.86.194): 56 data bytes
64 bytes from 10.10.86.194: seq=0 ttl=63 time=0.080 ms
64 bytes from 10.10.86.194: seq=1 ttl=63 time=0.105 ms
64 bytes from 10.10.86.194: seq=2 ttl=63 time=0.050 ms
64 bytes from 10.10.86.194: seq=3 ttl=63 time=0.055 ms

docker exec workload-A ping -c 4 workload-E //跨主机通讯同网络通讯
PING workload-E (10.10.48.65): 56 data bytes
64 bytes from 10.10.48.65: seq=0 ttl=62 time=0.584 ms
64 bytes from 10.10.48.65: seq=1 ttl=62 time=0.491 ms
64 bytes from 10.10.48.65: seq=2 ttl=62 time=0.843 ms
64 bytes from 10.10.48.65: seq=3 ttl=62 time=0.551 ms

docker exec workload-A ping -c 4 workload-B //同主机不同网络通讯
ping: bad address 'workload-B'

docker exec workload-A ping -c 4 workload-D //跨主机不同网络通讯
ping: bad address 'workload-D'
```

同时也可以查看到各容器的 ip 地址：
workload-A:cali0@if6:10.10.86.192/32
workload-B:cali0@if8:10.10.86.193/32
workload-C:cali0@if10:10.10.86.194/32
workload-D:cali0@if6:10.10.48.64/32
workload-E:cali0@if8:10.10.48.65/32

