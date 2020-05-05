# docker搭建prometheus、grafana监控Mysql

[toc]

## 简介

使用prometheus、grafana等搭建指标收集、存储、展示的监控系统。

![image-20200505141844766](/Users/zhoutaoo/Library/Application Support/typora-user-images/image-20200505141844766.png)

![image-20200505141917216](/Users/zhoutaoo/Library/Application Support/typora-user-images/image-20200505141917216.png)

mysql在宿主机上直接搭建，prometheus、grafana等都是通过docker来搭建的。

| 服务            | 启动方式 | 私网ip     | 端口      | 备注 |
| --------------- | -------- | ---------- | --------- | ---- |
| mysql           | VM       | 172.17.0.1 | 3306      |      |
| grafana         | docker   | 172.17.0.3 | 3000:3000 |      |
| prometheus      | docker   | 172.17.0.2 | 9090:9090 |      |
| mysqld-exporter | docker   | 172.17.0.1 | 9104      |      |
| node-exporter   | docker   | 172.17.0.1 | 9100      |      |

## prometheus搭建

### prometheus搭建

​		Prometheus是一套开源的监控&报警&时间序列数据库的组合，起始是由[SoundCloud](https://soundcloud.com/)公司开发的。随着发展，越来越多公司和组织接受采用Prometheus，社区也十分活跃，他们便将它独立成开源项目，并且有公司来运作。google SRE的书内也曾提到跟他们BorgMon监控系统相似的实现是Prometheus。现在最常见的Kubernetes容器管理系统中，通常会搭配Prometheus进行监控。

强大的多维度数据模型：

1. 时间序列数据通过 metric 名和键值对来区分。
2. 所有的 metrics 都可以设置任意的多维标签。
3. 数据模型更随意，不需要刻意设置为以点分隔的字符串。
4. 可以对数据模型进行聚合，切割和切片操作。
5. 支持双精度浮点类型，标签可以设为全 unicode。

使用docker启动prometheus

```shell
#将prometheus的配置文件映射到host，方便之后修改配置
docker run -d -p 9090:9090 -v ~/docker/prometheus/:/etc/prometheus/ prom/prometheus
```

配置文件 `~/docker/prometheus/prometheus.yml`

172.17.0.1为宿主机的docker私网ip。

```properties
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
scrape_configs:
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'server'
    static_configs:
    - targets: ['172.17.0.1:9100']
  - job_name: 'mysql'
    static_configs:
    - targets: ['172.17.0.1:9104']
```

### prometheus收集器搭建

​		监控exporter——监控当前机器自身的状态，包括硬盘、CPU、流量等。因为Prometheus已经有了很多现成的常用exporter。

​		所以我们直接用其中的node-exporter。在Prometheus看来，一台机器或者说一个节点就是一个node，所以该exporter是在上报当前节点的状态。mysqld-exporter是专门监控mysql，并将收集的数据提供给Prometheus。

exporter用host模式启动，与host共享ip，方便获取mysql的相关信息

```shell
#主机信息收集exporter
docker run -d \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter \
  --path.rootfs=/host
  
  #数据库信息收集exporter
  #DATA_SOURCE_NAME="用户名:密码@(mysqlip:port)"
  docker run -d \
  -p 9104:9104 \
  --net="host" \
  --pid="host" \
  -e DATA_SOURCE_NAME="root:Abc_12345678@(172.17.0.1:3306)/" \
  prom/mysqld-exporter
```

## grafana搭建

### grafana启动

```shell
docker run -d -p 3000:3000 grafana/grafana
```

### grafana配置

1. 配置prometheus数据源，url是prometheus的私网ip，Access选Server方式，因为grafana和prometheus都是docker来启动的。使用私网ip，通过服务端去获取数据。

   ![image-20200505150105760](/Users/zhoutaoo/Library/Application Support/typora-user-images/image-20200505150105760.png)

2. 导入grafana监控模板，mysql监控模板id为7362，填入如下输入框内

   ![image-20200505145701920](/Users/zhoutaoo/Library/Application Support/typora-user-images/image-20200505145701920.png)

   ![image-20200505145957342](/Users/zhoutaoo/Library/Application Support/typora-user-images/image-20200505145957342.png)

3.导入主机的监控模板，模板id为8919，以相同方式导入即可。