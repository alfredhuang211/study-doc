# docker跨主机overlay网络组建方法:

## 环境说明:
 
ubuntu14.04.4 
 
4.2.0 kernel
 
docker 1.10.3
 
etcd 2.3.6

## etcd配置:

### etcd启动

etcd启动命令:

```bash
etcd --advertise-client-urls 'http://192.168.15.232:2379' --listen-client-urls 'http://0.0.0.0:2379'
```

需要确保 advertise-client-urls 是在正确的ip和端口上监听

### etcd检查

* 本机检查:

在etcd运行的机器上,检查启动情况:

```bash
etcdctl member list
ce2a822cea30bfca: name=default peerURLs=http://localhost:2380,http://localhost:7001 clientURLs=http://192.168.15.232:2379 isLeader=true
```

检查功能是否正确,能否正确设置和获取
```bash
etcdctl mk key value
etcdctl get key 
value
```

* 远程检查

在其他主机上,验证远程连接的正确性,是否可以正确设置和获取
```bash
./etcdctl -endpoints http://192.168.15.232:2379 get key
value
/etcdctl -endpoints http://192.168.15.232:2379 mk key2 value2
value2
./etcdctl -endpoints http://192.168.15.232:2379 get key2 
value2
```


## docker配置

在/etc/docker下新建或修改文件daemon.json,添加内容:

一台主机

```json
{
    "cluster-store":"etcd://192.168.15.232:2379",
    "cluster-advertise":"192.168.15.232:2376"
}
```

另外一台主机

```json
{
    "cluster-store":"etcd://192.168.15.232:2379",
    "cluster-advertise":"192.168.15.233:2376"
}
```

其中cluster-store指向etcd所在设备ip,cluster-advertise内填写本机ip地址

### 配置生效

如果是1.10.3版本的docker,需要重启docker服务

```bash
service docker restart
```

如果是1.11版本的docker,可以通过发送信号reload配置文件,ps查看到docker daemon的pid:

```bash
kill -s SIGHUP 8182
```

### 生效检查

执行命令

```bash
docker info
```

其中信息有:

```
Cluster store: etcd://192.168.15.232:2379
Cluster advertise: 192.168.15.233:2376
```

说明集群信息正确配置进入了docker deamon

另外:

```
Plugins: 
 Volume: local
 Network: overlay bridge null host
```

此处也说明了支持创建的网络类型.其中overlay就是用来建立跨主机网络的类型.

## docker网络创建

执行命令:

```bash
docker network create -d overlay t1

docker network ls

NETWORK ID          NAME                DRIVER
f5ec4b3e24d8        t1                  overlay             
9a1aa00e5873        docker_gwbridge     bridge              
b71af588fbe1        none                null                
61fab2c54528        host                host                
507e83023697        bridge              bridge    
```

可以看到网络已建立.
可以在创建是使用 '--subnet=192.168.1.0/24' 这种类型来规定创建子网的网段.

### 确认创建

在其他主机上执行 'docker network ls',应该均能看到已创建的t1网络的信息.

## 容器加入

### 创建时加入

在执行 'docker run' 时使用参数 '--net t1', 使得创建的容器能加入t1网络:

```bash
docker run -id --net t1 --name t233_t1 busybox sh
```

### 创建后加入

已经在运行的容器也可以加入网络,通过执行命令:

```bash
docker run -id --name t233_t1 busybox sh
docker network connect t1 t233_out
```

## 网络互通

在不同主机上创建或加入容器到网络内,并在容器内互相 ping .在这种情况下,可以通过直接 ping 容器名的方式确认网络连接已通.

```bash
docker exec t233_t1 ping t232_t1
```

## 其他

* 在使用overlay网络的情况下,容器默认带有的bridge网络还是存在,可以在容器内看到两个网口.这种时候容器访问外网通过的是bridge网络.

```bash
docker exec 233_t1 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
79: eth0@if80: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue 
    link/ether 02:42:0a:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe00:2/64 scope link 
       valid_lft forever preferred_lft forever
81: eth1@if82: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:2/64 scope link 
       valid_lft forever preferred_lft forever
       
docker exec 233_t1 ping www.baidu.com
PING www.baidu.com (112.80.248.73): 56 data bytes
64 bytes from 112.80.248.73: seq=0 ttl=54 time=39.120 ms
64 bytes from 112.80.248.73: seq=1 ttl=54 time=40.778 ms
64 bytes from 112.80.248.73: seq=2 ttl=54 time=39.840 ms
```

* 容器可以加入多个overlay网络.但是在docker run的时候,只能通过 --net 指定一个网络,要加入其他网络需要在启动后通过 docker network connect 来添加



