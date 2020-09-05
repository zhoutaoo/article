# Docker进阶

## 网络模型

Docker 安装时会自动在 host 上创建三个网络：none、host、container和bridge，可通过如下命令查看。

```shell
[root@10-9-54-71 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9e3c441ad16e        bridge              bridge              local
fdfd4abed1a0        host                host                local
8a93bc235b8d        none                null                local
```

### bridge 模式

Docker 容器默认使用 bridge 模式的网络。其特点如下：

- 使用一个 linux bridge，默认为 docker0，docker0的linux bridge自动被创建好了，其上有一个 docker0 内部接口，IP地址为172.17.0.1/16

- 使用 veth 对，一头在容器的网络 namespace 中，一头在 docker0 上

- 该模式下Docker Container会有一个私网IP，宿主机的IP地址与veth pair的 IP地址不在同一个网段内

  

  ```shell
  [root@10-9-54-71 ~]# ip a
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1454 qdisc pfifo_fast state UP qlen 1000
      link/ether 52:54:00:ee:31:ea brd ff:ff:ff:ff:ff:ff
      inet 10.9.54.71/16 brd 10.9.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
      link/ether 02:42:2e:35:a5:67 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.1/16 scope global docker0
         valid_lft forever preferred_lft forever
  9: veth9303858@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
      link/ether 96:7f:a4:e3:de:d6 brd ff:ff:ff:ff:ff:ff link-netnsid 1
  17: veth66e2e72@if16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
      link/ether 86:c9:37:9b:01:f1 brd ff:ff:ff:ff:ff:ff link-netnsid 2
  ```

  ![docker-veth](/Users/zhoutaoo/Downloads/docker-bridge.jpg)

```shell
[root@10-9-54-71 ~]# brctl show
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02422e35a567	no		veth66e2e72
							                    veth9303858
```

### Host 模式

Host 模式并没有为容器创建一个隔离的网络环境。而之所以称之为host模式，是因为该模式下的 Docker 容器会和 host 宿主机共享同一个网络 namespace，故 Docker Container可以和宿主机一样，使用宿主机的eth0，实现和外界的通信。换言之，Docker Container的 IP 地址即为宿主机 eth0 的 IP 地址。其特点包括：

- 这种模式下的容器没有隔离的 network namespace
- 容器的 IP 地址同 Docker host 的 IP 地址
- 需要注意容器中服务的端口号不能与 Docker host 上已经使用的端口号相冲突
- host 模式能够和其它模式共存

```
docker run -d --network host -p 8080:80 nginx
```

![docker-host](/Users/zhoutaoo/Downloads/docker-host.jpg)

### container 模式

 Container 网络模式是 Docker 中一种较为特别的网络的模式。处于这个模式下的 Docker 容器会共享其他容器的网络环境，因此，至少这两个容器之间不存在网络隔离，而这两个容器又与宿主机以及除此之外其他的容器存在网络隔离。 

```shell
#（1）启动一个容器： 
docker run -d --name hostcs1 -p 8080:80 nginx
#（2）启动另一个容器，并使用第一个容器的 network namespace
docker run -d --name hostcs2 --network container:hostcs1  nginx
```

注意：因为此时两个容器要共享一个 network namespace，因此需要注意端口冲突情况，否则第二个容器将无法被启动。

示意图：

![docker-container](/Users/zhoutaoo/Downloads/docker-container.jpg)

### none 模式

网络模式为 none，即不为 Docker 容器构造任何网络环境。Docker Container的none网络模式容器内部就只能使用loopback网络设备，容器只能使用127.0.0.1的本机网络。

`docker run -d --network none nginx`

## 端口映射

docker容器在启动的时候，如果不指定端口映射参数，在容器外部是无法通过网络来访问容器内的网络应用和服务的。

可使用-p、-P来实现：

- -p指定要映射的端口，一个指定端口上只可以绑定一个容器
- -P将容器内部开放的网络端口随机映射到宿主机的一个端口上

```shell
# 将容器指定端口指定映射到宿主机的一个端口上
docker run -p 8080:80 -d  nginx
# 将容器指定端口随机映射到宿主机一个端口上。
docker run -P 80 -d nginx
```



![docker-port](/Users/zhoutaoo/Downloads/docker-port.jpg)

```shell
[root@10-9-54-71 ~]# iptables -t nat -vnL
Chain PREROUTING (policy ACCEPT 1763K packets, 107M bytes)
 pkts bytes target     prot opt in     out     source               destination
1723K  105M DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 1720K packets, 104M bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 1309K packets, 91M bytes)
 pkts bytes target     prot opt in     out     source               destination
 282K   17M DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 1028K packets, 75M bytes)
 pkts bytes target     prot opt in     out     source               destination
 325K   20M MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.3           172.17.0.3           tcp dpt:3000
    0     0 MASQUERADE  tcp  --  *      *       172.17.0.2           172.17.0.2           tcp dpt:9090

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
   12   720 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
 1068 57672 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:3000 to:172.17.0.3:3000
    0     0 DNAT       tcp  --  !docker0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9090 to:172.17.0.2:9090
```

## 跨主机通讯

多台host上的docker一般情况是不能进行通讯的，需要有网关相关支持才能通讯相互访问，主要有如下几种方案：

- 隧道方案：比如Flannel的 VxLan。特点是对底层的网络没有过高的要求，一般来说只要是三层可达就可以，只要是在一个三层可达网络里，就能构建出一个基于隧道的容器网络。问题也很明显，一个大家共识是随着节点规模的增长复杂度会提升，而且出了网络问题跟踪起来比较麻烦，大规模集群情况下这是需要考虑的一个点。 
- 路由方案：路由技术从三层实现跨主机容器互通，没有NAT，效率比较高，和目前的网络能够融合在一起，每一个容器都可以像虚拟机一样分配一个业务的IP。但路由网络也有问题，路由网络对现有网络设备影响比较大，路由器的路由表应该有空间限制一般是两三万条。而容器的大部分应用场景是运行微服务，数量集很大。如果几万新的容器IP冲击到路由表里，导致下层的物理设备没办法承受；而且每一个容器都分配一个业务IP，业务IP消耗会很快。  
- VLAN：所有容器和物理机在同一个 VLAN 中。

 Docker 多节点网络模式可以分为两类，一类是 Docker 在 1.19 版本中引入的基于 VxLAN 的对跨节点网络的原生支持；另一种是通过插件（plugin）方式引入的第三方实现方案，比如 Flannel，Calico 等等。

![zhaodongdb](/Users/zhoutaoo/Downloads/docker-h2h.jpg)

### Flannel容器网络

Flannel 是由 CoreOS 主导的解决方案。Flannel 为每一个主机的 Docker daemon 分配一个IP段，通过 etcd 维护一个跨主机的路由表，容器之间 IP 是可以互相连通的，当两个跨主机的容器要通信的时候，会在主机上修改数据包的 header，修改目的地址和源地址,经过路由表发送到目标主机后解包。封包的方式，可以支持udp、vxlan、host-gw等，但是如果一个容器要暴露服务，还是需要映射IP到主机侧的。

### Calico 网络方案

Calico 是个年轻的项目，基于BGP协议，完全通过三层路由实现。Calico 可以应用在虚机，物理机，容器环境中。在Calico运行的主机上可以看到大量由 linux 路由组成的路由表，这是calico通过自有组件动态生成和管理的。这种实现并没有使用隧道，没有NAT，导致没有性能的损耗，性能很好，从技术上来看是一种很优越的方案。这样做的好处在于，容器的IP可以直接对外部访问，可以直接分配到业务IP，而且如果网络设备支持BGP的话，可以用它实现大规模的容器网络。但BGP带给它的好处的同时也带给他的劣势，BGP协议在企业内部还很少被接受，企业网管不太愿意在跨网络的路由器上开启BGP协议。

### Docker 网络模式选择的简单结论

- Bridge 模式的性能损耗大概为10%
- 原生 overlay 模式的性能损耗非常高，甚至达到了 56%，因此，在生产环境下使用这种模式需要非常谨慎。
- 如果一定要使用 overlay 模式的话，可以考虑使用 Cisco 发起的  Calico 模式，它的性能和 bridge 相当。
- Weave overlay 模式的性能数据非常可疑，按理说应该不可能这么差。

## Volume使用

为了很好的实现数据保存和数据共享，Docker提出了***Volume***这个概念，简单的说就是绕过默认的联合文件系统，而以正常的文件或者目录的形式存在于宿主机上。又被称作***数据卷***。

### Volume的作用

- 通过数据卷可以在容器之间实现共享和重用
- 对数据卷的修改会立马生效(非常适合作为开发环境)
- 对数据卷的更新,不会影响镜像
- 卷会一直存在，直到没有容器使用

### Docker volume

容器映射Host目录

`docker run -p 8080:80 -d -v /usr/share/nginx/html nginx`

创建数据卷

```
docker volume create --name dbstore
docker run -d --volumes-from dbstore nginx
```

### Flocker：容器分布式存储平台

- 容器的数据会被写入 Flocker 后端存储而不是主机上，因此，在主机出现故障时可以保证数据不丢失
- 在容器迁移时，Flocker 会自动地将卷从一个 host 移植到另一个 host



