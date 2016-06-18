# docker macvlan 跨主机网络操作方法

## 搭建环境

* virtualbox, ubuntu14.04.4 内核4.2.0 docker 1.12.0-rc1

* virtualbox上运行两套主机系统,设置使用桥接模式,网卡混杂模式开启全部允许.

* 主机上配置的eth0网口或者创建的vlan网口,均需要开启混杂模式,命令 `ip link set eth0 promisc on` `ip link set eth0.100 promisc on`

 **注意** : 如果不开启混杂模式,会导致macvlan网络无法访问外界,具体在不使用vlan时,表现为无法ping通路由,无法ping通同一网络内其他主机

## 搭建过程1-不使用vlan

1. 创建docker macvlan网络

 两台主机上 eth0 使用分别为 192.168.15.75/192.168.15.21. 分别在两台主机上使用相同命令 `docker network create -d macvlan --subnet=192.168.15.0/24 --gateway=192.168.15.1 -o parent=eth0 -o macvlan_mode=bridge eth0_1` 创建eth0_1的macvlan网络.

2. 创建容器

 主机1 运行容器 使用命令:

 `docker run --net=eth0_1 --ip=192.168.15.101 -id --name test101 busybox sh`

 `docker run --net=eth0_1 --ip=192.168.15.102 -id --name test102 busybox sh`

 主机2 运行容器 使用命令:

 `docker run --net=eth0_1 --ip=192.168.15.201 -id --name test201 busybox sh`

 `docker run --net=eth0_1 --ip=192.168.15.202 -id --name test202 busybox sh`

3. 测试网络

 主机1上测试:

 运行命令:

 `docker exec test101 ping 192.168.15.1` ping网关: **通**

 `docker exec test101 ping test102` 使用容器名ping本主机容器: **通**

 `docker exec test102 ping 192.168.15.101` ping本主机容器: **通**

 `docker exec test101 ping 192.168.15.201` ping另一主机容器: **通**

 `docker exec test101 ping test201` 使用容器名ping另一主机容器: **不通**

 `ping 192.168.15.101` 本主机ping本主机容器: **不通**

 `ping 192.168.15.201` 本主机ping另一主机容器: **通**

 主机2上测试获取相同结果.

## 搭建过程2-使用vlan

1. 创建vlan

 使用命令`vconfig add eth0 100` 创建eth0.100的vlan.设置两台主机的vlan ip分别为192.168.100.50/192.168.100.51

2. 创建docker macvlan网络

 分别在两台主机上使用命令 `docker network create -d macvlan --subnet=192.168.100.0/24 --gateway=192.168.100.1 -o parent=eth0.100 -o macvlan_mode=bridge 100_1` 创建相同的100_1的macvlan网络.

3. 创建容器

 主机1 运行容器 使用命令:

 `docker run --net=100_1 --ip=192.168.100.101 -id --name test100.101 busybox sh`

 `docker run --net=100_1 --ip=192.168.100.102 -id --name test100.102 busybox sh`

 主机2 运行容器 使用命令:

 `docker run --net=100_1 --ip=192.168.100.201 -id --name test100.201 busybox sh`

 `docker run --net=100_1 --ip=192.168.100.202 -id --name test100.202 busybox sh`

4. 测试网络

 主机1上测试:

 运行命令:

 `docker exec test100.101 ping 192.168.100.1` ping网关: **不通**

 `docker exec test100.101 ping 192.168.100.50` ping本地eth0.100地址: **不通**

 `docker exec test100.101 ping 192.168.100.51` ping另一个主机的eth0.100地址:**通**

 `docker exec test100.101 ping 192.168.100.102` ping本主机容器: **通**

 `docker exec test100.101 ping 192.168.100.201` ping另一主机容器: **通**

 `docker exec test100.101 ping test100.102` 使用容器名ping本主机容器: **通**

 `docker exec test100.101 ping test100.201` 使用容器名ping另一主机容器: **不通**

 `ping 192.168.100.101` 本主机ping本主机容器: **不通**

 `ping 192.168.100.201` 本主机ping另一主机容器: **通**

 主机2测试相同
