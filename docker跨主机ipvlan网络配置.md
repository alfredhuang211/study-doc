# docker跨主机ipvlan网络配置

## 说明

ipvlan的网络配置很类似与macvlan,见 [docker跨主机macvlan网络配置](docker跨主机macvlan网络配置.md)

## 搭建环境

* virtualbox, ubuntu14.04.4 内核4.2.0 docker 1.12.0-rc2

* virtualbox上运行两套主机系统,设置使用桥接模式,网卡混杂模式开启全部允许.

**注意** : 不用类似于macvlan,在主机上的网口,无需开启混杂模式

## 搭建过程1-不使用vlan

### 1. 创建docker ipvlan网络

两台主机上 eth0 使用分别为 192.168.4.236/192.168.4.237. 分别在两台主机上使用相同命令 `docker network create -d ipvlan --subnet=192.168.4.0/24 --gateway=192.168.4.1 -o parent=eth0 -o ipvlan_mode=l2 eth0_1` 创建eth0_1的ipvlan网络.

### 2. 创建容器

主机1 运行容器 使用命令:

`docker run --net=eth0_1 --ip=192.168.4.101 -id --name test101 busybox sh`

`docker run --net=eth0_1 --ip=192.168.4.102 -id --name test102 busybox sh`

主机2 运行容器 使用命令:

`docker run --net=eth0_1 --ip=192.168.4.201 -id --name test201 busybox sh`

`docker run --net=eth0_1 --ip=192.168.4.202 -id --name test202 busybox sh`

### 3. 测试网络

主机1上测试:

运行命令:

`docker exec test101 ping 192.168.4.1` ping网关: **通**

`docker exec test101 ping test102` 使用容器名ping本主机容器: **通**

`docker exec test102 ping 192.168.4.101` ping本主机容器: **通**

`docker exec test102 ping 192.168.4.199` ping本网络其他主机: **通**

`docker exec test101 ping 192.168.4.201` ping另一主机容器: **通**

`docker exec test101 ping test201` 使用容器名ping另一主机容器: **不通**

`ping 192.168.4.101` 本主机ping本主机容器: **不通**

`ping 192.168.4.201` 本主机ping另一主机容器: **通**

主机2上测试获取相同结果.

## 搭建过程2-使用vlan

### 1. 创建vlan

使用命令`vconfig add eth0 100` 创建eth0.100的vlan.设置两台主机的vlan ip分别为192.168.15.50/192.168.15.51

### 2. 创建docker ipvlan网络

分别在两台主机上使用命令 `docker network create -d ipvlan --subnet=192.168.100.0/24 --gateway=192.168.100.1 -o parent=eth0.100 -o ipvlan_mode=l2 100_1` 创建相同的100_1的ipvlan网络.

### 3. 创建容器

主机1 运行容器 使用命令:

`docker run --net=100_1 --ip=192.168.100.101 -id --name test100.101 busybox sh`

`docker run --net=100_1 --ip=192.168.100.102 -id --name test100.102 busybox sh`

主机2 运行容器 使用命令:

`docker run --net=100_1 --ip=192.168.100.201 -id --name test100.201 busybox sh`

`docker run --net=100_1 --ip=192.168.100.202 -id --name test100.202 busybox sh`

### 4. 测试网络

主机1上测试:

运行命令:

`docker exec test100.101 ping 192.168.100.1` ping网关: **不通**

`docker exec test100.101 ping 192.168.15.50` ping本地eth0.100地址: **不通**

`docker exec test100.101 ping 192.168.15.51` ping另一个主机的eth0.100地址: **不通**

`docker exec test100.101 ping 192.168.100.102` ping本主机容器: **通**

`docker exec test100.101 ping 192.168.100.201` ping另一主机容器: **通**

`docker exec test100.101 ping test100.102` 使用容器名ping本主机容器: **通**

`docker exec test100.101 ping test100.201` 使用容器名ping另一主机容器: **不通**

`docker exec test100.101 ping 192.168.4.199` ping跨网络主机: **不通**

`ping 192.168.100.101` 本主机ping本主机容器: **不通**

`ping 192.168.100.201` 本主机ping另一主机容器: **不通**

主机2测试相同

## 一些问题

ipvlan网络在创建时要指定parent.其中parent仅能使用一次,即eth0在创建一个网络时使用了,则在创建另一个的时候就无法再使用了.

ipvlan网络的子网不能和vlan网络的网段相同,如果相同反而导致ipvlan网络内的容器无法跨主机通讯.

在创建ipvlan的时候,如果不指定网段,默认网段为172.18.0.0/16, 此时加入此网络的容器,在同一台主机上时可以互相ping通,无法ping通外部网络,同时加入此网络的容器,创建时不能手工指定ip,仅能使用ipam自动分配.

**在使用和主机相同网段的ipvlan时,如果在创建容器时不指定ip,则默认ipam从192.168.4.2开始分配,不检查网段内是否已有相同ip,这种情况下会造成容器ip和网络内其他设备ip冲突.**
