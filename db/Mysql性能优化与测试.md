## Mysql性能优化与测试

### 操作系统优化

```shell
# 修改系统 进程的最大数目 和 最大打开的文件数
vim /etc/security/limits.conf  
* soft nproc 65535 
* hard nproc 65535 
* soft nofile 65535 
* hard nofile 65535

# 查看 Linux内核参数调优
vim /etc/sysctl.conf 
# 用户端口范围 
net.ipv4.ip_local_port_range = 1024 65535：
net.ipv4.tcp_max_syn_backlog = 4096  
net.ipv4.tcp_fin_timeout = 30  
# 系统文件句柄，控制的是能打开文件数量
fs.file-max=65535
# Swap 调整(不使用 swap 分区)：
vm.swappiness=0
```

### Mysql服务端调优

#### 连接数与线程

```shell
# 指定MySQL为快速重用而缓存的线程数量 
thread_cache_size： #值的范围从0~16384，默认值为0。根据内存大小设置1G→8，2G→16，3G→32，大于3G→64
thread_stack：      #每个线程堆栈大小，默认256K。

# 连接相关
max_connections           # 实例最大连接数，根据内存设置合理值     
max_connect_errors        # 错误连接数，能大则大 
max_user_connections      # 用户级最大连接数，0为不限制 
connect_timeout           # 连接超时 
skip-name-resolve         # 跳过域名解析 
wait_timeout              # 等待超时 
back_log                  # 可以在堆栈中的连接数量 

# 查看当前mysql线程相关数据
show global status like 'Threads_%';

Threads_cached    : 当前线程池中缓存有多少空闲线程
Threads_connected : 当前的连接数 ( 也就是线程数 )
Threads_created   : 已经创建的线程总数
Threads_running   : 当前激活的线程数 ( Threads_connected 中的线程有些可能处于休眠状态 )
```

#### 内存缓冲区

MySQL占用内存 = 全局缓存  + ( 线程缓存 x 最大连接数 )

```shell
# 全局缓存，用于判断内存参数设置是否合理
key_buffer_size:         #索引缓存区大小，对于内存在4GB左右的服务器该参数可设置为384M或512M
innodb_buffer_pool_size：#缓存InnoDB的索引及数据，至关重要的参数，但是尽量设置不要超过物理内存70%
innodb_log_buffer_size： #InnoDB事务日志使用的缓冲区，100M以下
max_heap_table_size：    #用户可以创建的内存表大小
query_cache_size：       #缓存查询(SELECT)的结果

SELECT (@@key_buffer_size + @@innodb_buffer_pool_size + @@innodb_log_buffer_size + @@max_heap_table_size + @@query_cache_size)/1024/1024;

# 索引缓存
SHOW GLOBAL STATUS LIKE '%key_read%';
+-------------------+-----------------+
| Variable_name     | Value           |
+-------------------+-----------------+
| Key_read_requests | 2454354135490   |
| Key_reads         | 23490           |
+-------------------+-----------------+
# 一共有Key_read_requests个索引请求，一共发生了Key_reads次物理IO。Key_reads/Key_read_requests ≈ 0.1% 以下比较好。
SHOW GLOBAL STATUS LIKE 'key_blocks_u%';
+------------------------+-------------+
| Variable_name          | Value       |
+------------------------+-------------+
| Key_blocks_unused      | 0           |
| Key_blocks_used        | 413543      |
+------------------------+-------------+
# Key_blocks_unused表示未使用的缓存簇(blocks)数，Key_blocks_used表示曾经用到的最大的blocks数，比较理想的设置：Key_blocks_used / (Key_blocks_unused + Key_blocks_used) * 100% ≈ 80%

# 用户级buffer参数
max_allowed_packet    #数据包发送缓冲区是存储接传送数据包的内存区域
thread_stack          #每个线程堆栈大小，默认256K。
sort_buffer_size      #连接排序缓冲区内存大小
join_buffer_size      #连接使用连接缓冲区大小
read_buffer_size      #全表扫描时分配的缓冲区大小
read_rnd_buffer_size  #随机读取缓存 
myisam_sort_buffer_size  #myisam使用的排序缓冲区大小，一般不使用myisam

# 线程缓存 ＝ sort_buffer_size ＋ read_rnd_buffer_size ＋ join_buffer_size ＋ read_buffer_size ＋ max_allowed_packet ＋ thread_stack
                
SELECT (@@max_allowed_packet + @@thread_stack + @@read_rnd_buffer_size + @@read_buffer_size + @@sort_buffer_size + @@join_buffer_size)/1024/1024 * @@max_connections;
```

| **物理内存**        | 1G   | 2G   | 4G   | 8G    | 16G   |
| ------------------- | ---- | ---- | ---- | ----- | ----- |
| **key_buffer_size** | 128M | 256M | 384M | 1024M | 2048M |

#### Innodb引擎相关

```shell
# innodb引擎相关
innodb_file_per_table=(1,0)   # 1为每个表一个单独文件，一般都选择1。
innodb_flush_log_at_trx_commit=(0,1,2) # 1是最安全的，0是性能，2折中 
Innodb_flush_method=(O_DIRECT, 默认fdatasync) 
innodb_log_file_size          # 100M以下 
innodb_log_files_in_group     # 5个成员以下,一般2-3个够用（iblogfile0-N） 
innodb_max_dirty_pages_pct    # 达到百分之75的时候刷写 内存脏页到磁盘。 
max_binlog_cache_size         # 可以不设置 
max_binlog_size               # 可以不设置 
#小于2G内存的机器，推荐值是20M。32G内存以上100M 
innodb_additional_mem_pool_size    

explicit_defaults_for_timestamp=true
# bin log过期清理时间，单位天
expire_logs_days=30
#binlog格式为row模式
binlog_format=row
#允许下端接入slave
log_slave_updates=1
#开启GTID
gtid_mode=on
#启用强一致性检查，避免create table...select操作
enforce_gtid_consistency=on
```

### Msql性能测试

#### 测试过程

```shell
# 安装测试软件
yum install -y sysbench

# 1.先创建数据
mysql> create database sbtest;

# 2.造数32线程10表
sysbench /usr/share/sysbench/oltp_common.lua --time=300 --mysql-host=10.9.54.71 --mysql-port=3306 --mysql-user=sbux_cac_data --mysql-password=Abc@12345 --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=32 --events=999999999  prepare
# 3.执行测试，32线程10表测试
sysbench /usr/share/sysbench/oltp_read_write.lua --time=300 --mysql-host=10.9.54.71 --mysql-port=3306 --mysql-user=sbux_cac_data --mysql-password=Abc@12345 --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=32 --events=999999999  --report-interval=10  run
# 清理数据
sysbench /usr/share/sysbench/oltp_read_write.lua --time=300 --mysql-host=10.9.54.71 --mysql-port=3306 --mysql-user=sbux_cac_data --mysql-password=Abc@12345 --mysql-db=sbtest --table-size=1000000 --tables=10 --threads=32 --events=999999999  --report-interval=10  cleanup
```

#### 测试结果

```
SQL statistics:
    queries performed:
        read:                            2276540
        write:                           650440
        other:                           325220
        total:                           3252200
    transactions:                        162610 (541.90 per sec.)
    queries:                             3252200 (10837.92 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          300.0740s
    total number of events:              162610

Latency (ms):
         min:                                    8.28
         avg:                                   59.03
         max:                                  480.86
         95th percentile:                      116.80
         sum:                              9599333.16

Threads fairness:
    events (avg/stddev):           5081.5625/108.21
    execution time (avg/stddev):   299.9792/0.02
```

