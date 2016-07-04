# docker swarm mode 配置使用

## swarm mode说明

swarm mode 是 docker 在1.12版本中引入的集群配置工具,将原本的 docker swarm 集群配置工具直接集成在了 docker-engine 中.这种情况下,在只安装了docker-engine的情况下,都可以完成 docker 集群的组建和相应的操作.

## 运行环境

ubuntu14.04.4

4.2.0 kernel

docker 1.12.0-rc2 (注意:在docker 1.12.0-rc1后才有swarm mode)

准备了ip分别为192.168.4.231/232/233的3台主机

此时查看 `docker info` 中, Swarm状态为inactive.

## 创建集群

在主机231上执行 `docker swarm init --listen-addr 192.168.4.231:2377`

此时通过 `docker info` 查看 Swarm 状态如下:
```bash
Swarm: active
 NodeID: 1lsp4r0esfl6a7vtg5hb02ztm
 IsManager: Yes
 Managers: 1
 Nodes: 1
 CACertHash: sha256:63030424e110733683d6f7543e3887f12021a1704592960e492a170144c025f5
```

通过 `docker node ls` 查看如下:
```bash
root@ubuntu231:~# docker node ls
ID                           NAME       MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
1lsp4r0esfl6a7vtg5hb02ztm *  ubuntu231  Accepted    Ready   Active        Leader
```

通过 `docker swarm inspect` 查看如下:
```bash
root@ubuntu231:~# docker swarm inspect
[
    {
        "ID": "6toicfhqgbbjucri68f2pcgs2",
        "Version": {
            "Index": 11
        },
        "CreatedAt": "2016-07-04T12:07:56.816450141Z",
        "UpdatedAt": "2016-07-04T12:07:56.996227642Z",
        "Spec": {
            "Name": "default",
            "AcceptancePolicy": {
                "Policies": [
                    {
                        "Role": "worker",
                        "Autoaccept": true
                    },
                    {
                        "Role": "manager",
                        "Autoaccept": false
                    }
                ]
            },
            "Orchestration": {
                "TaskHistoryRetentionLimit": 10
            },
            "Raft": {
                "SnapshotInterval": 10000,
                "LogEntriesForSlowFollowers": 500,
                "HeartbeatTick": 1,
                "ElectionTick": 3
            },
            "Dispatcher": {
                "HeartbeatPeriod": 5000000000
            },
            "CAConfig": {
                "NodeCertExpiry": 7776000000000000
            }
        }
    }
]
```

## 节点加入集群

在232/233两主机上执行 `docker swarm join 192.168.4.231:2377`

此时两主机上查看 `docker info`:
```bash
Swarm: active
 NodeID: f51y1qkkarppreo9kt93f4zuy
 IsManager: No
```

此时两主机上查看 `docker node ls`:
```bash
root@ubuntu233:~# docker node ls
Error response from daemon: This node is not participating as a Swarm manager
```

在231上查看 `docker info`:
```bash
Swarm: active
 NodeID: 1lsp4r0esfl6a7vtg5hb02ztm
 IsManager: Yes
 Managers: 1
 Nodes: 3
 CACertHash: sha256:63030424e110733683d6f7543e3887f12021a1704592960e492a170144c025f5
```

在231上查看 `docker node ls`:
```bash
root@ubuntu231:~# docker node ls
ID                           NAME       MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
1lsp4r0esfl6a7vtg5hb02ztm *  ubuntu231  Accepted    Ready   Active        Leader
6dd63h58bw9vhuxp94208kq0a    ubuntu233  Accepted    Ready   Active        
f51y1qkkarppreo9kt93f4zuy    ubuntu232  Accepted    Ready   Active        
```

在231上查看 `docker swarm inspect`:
```bash
root@ubuntu231:~# docker swarm inspect
[
    {
        "ID": "6toicfhqgbbjucri68f2pcgs2",
        "Version": {
            "Index": 11
        },
        "CreatedAt": "2016-07-04T12:07:56.816450141Z",
        "UpdatedAt": "2016-07-04T12:07:56.996227642Z",
        "Spec": {
            "Name": "default",
            "AcceptancePolicy": {
                "Policies": [
                    {
                        "Role": "worker",
                        "Autoaccept": true
                    },
                    {
                        "Role": "manager",
                        "Autoaccept": false
                    }
                ]
            },
            "Orchestration": {
                "TaskHistoryRetentionLimit": 10
            },
            "Raft": {
                "SnapshotInterval": 10000,
                "LogEntriesForSlowFollowers": 500,
                "HeartbeatTick": 1,
                "ElectionTick": 3
            },
            "Dispatcher": {
                "HeartbeatPeriod": 5000000000
            },
            "CAConfig": {
                "NodeCertExpiry": 7776000000000000
            }
        }
    }
]
```

可以看到 231 主机为 manager 节点,其他两个为 worker 节点.在 worker 节点上,无法查看 swarm node 信息.

## 通过服务使用集群

在231,即 manager 节点上,通过执行如下命令启动一个服务.

`docker service create --replicas 3 --name test busybox ping www.baidu.com
`

执行后通过 `docker service ls` 查看服务情况.
```bash
root@ubuntu231:~# docker service ls
ID            NAME  REPLICAS  IMAGE    COMMAND
52mpkhlszznu  test  3/3       busybox  ping www.baidu.com
```

可以看到此服务复制数为3.

分别在3台主机查看容器

```bash
root@ubuntu231:~# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS               NAMES
4c2c3d36ee0d        busybox:latest      "ping www.baidu.com"   About a minute ago   Up About a minute                       test.1.bx4hz20s11msmlp04w3uerg18
```

```bash
root@ubuntu232:~# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS               NAMES
5fb24cf0ff0e        busybox:latest      "ping www.baidu.com"   About a minute ago   Up About a minute                       test.3.358cfwsw2ty2brt5ve045r61b

```

```bash
root@ubuntu233:~# docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS               NAMES
104191844da9        busybox:latest      "ping www.baidu.com"   About a minute ago   Up About a minute                       test.2.971p65qenkoqdo6hpvkq9f5id
```

可以看到此服务的容器被平均分布到3台主机上去运行了.

也可以通过在 manager 节点上执行 `docker service tasks test` 查看此服务任务的分布情况.

```bash
root@ubuntu231:~# docker service tasks test
ID                         NAME    SERVICE  IMAGE    LAST STATE         DESIRED STATE  NODE
bx4hz20s11msmlp04w3uerg18  test.1  test     busybox  Running 5 minutes  Running        ubuntu231
971p65qenkoqdo6hpvkq9f5id  test.2  test     busybox  Running 5 minutes  Running        ubuntu233
358cfwsw2ty2brt5ve045r61b  test.3  test     busybox  Running 5 minutes  Running        ubuntu232
```

在使用 service 服务的时候,每台主机上的 docker run 还是可以一样正常使用.

## 服务规模调整

在 manager 节点上执行命令 `docker service scale test=5` 即可完成服务扩容.此处是将test 服务的容器数扩容到 5.

通过 `docker service tasks test` 查看服务:

```bash
root@ubuntu231:~# docker service tasks test
ID                         NAME    SERVICE  IMAGE    LAST STATE          DESIRED STATE  NODE
bx4hz20s11msmlp04w3uerg18  test.1  test     busybox  Running 15 minutes  Running        ubuntu231
971p65qenkoqdo6hpvkq9f5id  test.2  test     busybox  Running 15 minutes  Running        ubuntu233
358cfwsw2ty2brt5ve045r61b  test.3  test     busybox  Running 15 minutes  Running        ubuntu232
b1cu1mwnw98oymi1vgbcl3l84  test.4  test     busybox  Running 20 seconds  Running        ubuntu232
7t8kfpfv4zegut9fbc1bbwaf6  test.5  test     busybox  Running 20 seconds  Running        ubuntu231
```

可以看到在经过一段时间后,此服务的容器数扩展到5个,并调度到不同主机上.

同理, 通过 `docker service scale test=2` 此命令指定不同的 scale 数,可以随意控制服务的容器数量.

```bash
root@ubuntu231:~# docker service tasks test
ID                         NAME    SERVICE  IMAGE    LAST STATE          DESIRED STATE  NODE
358cfwsw2ty2brt5ve045r61b  test.3  test     busybox  Running 17 minutes  Running        ubuntu232
7t8kfpfv4zegut9fbc1bbwaf6  test.5  test     busybox  Running 2 minutes   Running        ubuntu231
```

缩容的话, docker 会随机选择容器关闭.

## 服务移除

通过 `docker service rm test` 即可移除服务. 当移除服务后, 每台主机上相关的容器也就被移除了.