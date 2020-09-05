

# centos下安装mysql主从架构（半同步/多实例）

[toc]

## 简介

本教程会进行mysql一机多实例的安装、mysql主从同步配置、半同步配置

## 环境

OS: CentOS Linux release 7.2.1511 (Core)

mysql: mysql-5.7.29-linux-glibc2.12-x86_64

## mysql搭建

### 1.准备工作
```shell
# centos添加mysql用户
useradd -s /sbin/nologin mysql
# 创建工作目录
mkdir /var/workspace
cd workspace
# 下载mysql
wget http://mirrors.sohu.com/mysql/MySQL-5.7/mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz
# 解压mysql
tar -zxvf mysql-5.7.29-linux-glibc2.12-x86_64.tar.gz
# 修改文件属主为mysql用户
chown -R mysql:mysql mysql-5.7.29-linux-glibc2.12-x86_64
```

### 2.依赖安装
`yum -y install make gcc-c++ cmake bison-devel ncurses-devel libaio libaio-devel perl-Data-Dumper net-tools `

### 3.初使化
```shell
# 创建数据库数据目录
mkdir -p /var/workspace/mysql_3306/data
# 将数据目录属主改为mysql
chown -R mysql:mysql /var/workspace/mysql_3306
# 执行初使化
/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/bin/mysqld --initialize --user=mysql --basedir=/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/ --datadir=/var/workspace/mysql_3306/data --explicit_defaults_for_timestamp
```

> 注意：初使化时会随机生成一个root账号的密码，请记下来

### 4.主库配置

创建mysql主库的配置文件 /var/workspace/mysql_3306/data/my.cnf

```bash
[mysqld]
# mysql安装目录
basedir=/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64
# mysql数据存放目录
datadir=/var/workspace/mysql_3306/data
# mysql启动的用户
user=mysql
port=3306
socket=/tmp/mysql3306.sock
# 表名不区分大小写
lower_case_table_names=1
```

### 5.从库安装与配置

从库可安装在其它主机或同主机（端口与相关文件不能相同），目前配置文件相同。

### 6.启动mysql

#### 命令行启动

```shell
# 手工启动
/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/bin/mysqld_safe --defaults-file=/var/workspace/mysql_3306/data/my.cnf --pid-file=/var/workspace/mysql_3306/data/mysql_3306.pid >/dev/null &
# 通过 mysql.server 指定配置文件启动
/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/support-files/mysql.server --defaults-file=/var/workspace/mysql_3306/data/my.cnf --datadir=/var/workspace/mysql_3306/data --pid-file=/var/workspace/mysql_3306/data/mysql_3306.pid
```

#### 多实例启动

mysql多实例启动有两种：

1. 在/etc/my.cnf中配置多个mysql实例，通过mysqld_multi.server脚本启动，其中实例变化都要修改同一个配置文件

2. 通过mysql.server脚本启动，指定不同配置文件启动。为了简化启动，我修改了mysql.server脚本，如下

   `cp /var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/support-files/mysql.server /var/workspace/mysql.server`

   修改/var/workspace/mysql.server脚本如下位置和内容：

   ```bash
   # 46行47，修改 basedir和datadir路径
   basedir=/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64
   datadir=/var/workspace/mysql_$2/data
   # 266，增加默认配置文件路径，默认在数据目录下my.cnf
   $bindir/mysqld_safe --defaults-file="$datadir"/my.cnf --datadir="$datadir" --pid-file="$mysqld_pid_file_path"  >/dev/null &
   ```

   修改脚本后启动命令变为如下，第二个参数为数据目录mysql_后的值，一般为端口号，方便辨认

```bash
# 启动
/var/workspace/mysql.server start 3306
# 重启
/var/workspace/mysql.server restart 3306
# 停止
/var/workspace/mysql.server stop 3306
```

### 7.数据库验证

```mysql
# 本地登陆mysql，需要输入刚才记录下的密码
/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/bin/mysql -S /tmp/mysql3306.sock -uroot -p
# 查看数据库，需要重设置root密码
mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
# 设置root账号密码
mysql> set password='12345678';
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
mysql> create database db;
```

### 8.常用mysql命令

```mysql
show tables或show tables from database_name; -- 显示当前数据库中所有表的名称。
show databases; -- 显示mysql中所有数据库的名称。 
show columns from table_name from database_name;  -- 显示表中列名称。
show grants for user_name; -- 显示一个用户的权限，显示结果类似于gr
show columns from database_name.table_name; -- 显示表中列名称。
show grants for user_name; -- 显示一个用户的权限，显示结果类似于grant 命令。
show index from table_name; -- 显示表的索引。
show status; -- 显示一些系统特定资源的信息，例如，正在运行的线程数量。
show variables; -- 显示系统变量的名称和值。
show processlist; -- 显示系统中正在运行的所有进程，也就是当前正在执行的查询。大多数用户可以查看他们自己的进程，但是如果他们拥有process权限，就可以查看所有人的进程，包括密码。
show table status; -- 显示当前使用或者指定的database中的每个表的信息。信息包括表类型和表的最新更新时间。
show privileges; -- 显示服务器所支持的不同权限。
show create database database_name; -- 显示create database 语句是否能够创建指定的数据库。
show create table table_name; -- 显示create database 语句是否能够创建指定的数据库。
show engines; -- 显示安装以后可用的存储引擎和默认引擎。
show logs; -- 显示BDB存储引擎的日志。
show warnings; -- 显示最后一个执行的语句所产生的错误、警告和通知。
show errors; -- 只显示最后一个执行语句所产生的错误。
show engine innodb status\G; -- 显示innoDB存储引擎的状态。
show plugins; -- 查看mysql插件库
```

## mysql主从同步配置

### 主库配置

#### 1.修改配置文件

修改/var/workspace/mysql_3306/data/my.cnf文件，在[mysqld]下新增内容如下，修改后重启。

```bash
# 主数据库端ID号
server_id=1
# 开启binlog日志
log-bin=mysql-bin
# 需要复制的数据库名，如果复制多个数据库，重复设置这个选项即可
binlog-do-db=db
binlog-do-db=sbux
# 将函数复制到slave
log_bin_trust_function_creators=1
```

#### 2.创建slave同步账号

```mysql
# 创建slave账号，用于从库连接主库
mysql> grant replication slave,replication client on *.* to 'slave'@'10.9.62.185' identified by 'Abc@12345';
Query OK, 0 rows affected, 1 warning (0.00 sec)
# 刷新权限，使权限立即生效
mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)
# 查看主库状态，如下说明主库binlog已正常打开
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 3392
     Binlog_Do_DB: db,sbux
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

ERROR:
No query specified
```

### 从库配置

#### 1.修改配置文件

修改/var/workspace/mysql_3306/data/my.cnf文件，在[mysqld]下新增内容如下，修改后重启。

```bash
# 从节点的ID号,不能和主节点的一样
server_id=2
# 开启重放日志
relay_log=relay-log
# 从库只读开启
read_only=ON
```

#### 2.连接到主

```mysql
# 本地登陆mysql
/var/workspace/mysql-5.7.29-linux-glibc2.12-x86_64/bin/mysql -S /tmp/mysql3306.sock -uroot -p
# 连接主库，MASTER_HOST为主库ip，使用在主库创建的账号密码slave连接，MASTER_LOG_FILE为主库的log文件名
mysql> CHANGE MASTER TO MASTER_HOST='10.9.54.71',MASTER_USER='slave',MASTER_PASSWORD='Abc@12345',MASTER_LOG_FILE='mysql-bin.000001',MASTER_LOG_POS=0;
mysql> CHANGE MASTER TO MASTER_HOST='10.9.54.71',MASTER_USER='slave',MASTER_PASSWORD='Abc@12345',MASTER_AUTO_POSITION=1;
# 启动slave
mysql> START SLAVE;
# 查看从库slave状态，如下Slave_IO_State: Waiting for master to send event，说明从库下在等待主库事件。
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.9.54.71
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 3392
               Relay_Log_File: relay-log.000003
                Relay_Log_Pos: 3605
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 3392
              Relay_Log_Space: 3806
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: 2850977a-53f9-11ea-9c76-525400ee31ea
             Master_Info_File: /var/workspace/mysql_3306/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)

ERROR:
No query specified
```

### 数据验证

#### 1.主库操作

```mysql
mysql> use db;
mysql> CREATE TABLE sbux_users (
       id BIGINT ( 20 ) NOT NULL,
       `name` VARCHAR ( 20 ) NOT NULL,
       `desc` VARCHAR ( 100 ),
       PRIMARY KEY ( id )
       ) ENGINE = INNODB DEFAULT CHARSET = utf8mb4;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO sbux_users(id,`name`,`desc`) VALUES(1,'zhangs','goods1');
Query OK, 1 row affected (0.00 sec)
mysql> INSERT INTO sbux_users(id,`name`,`desc`) VALUES(2,'zhangs2','goods2');
Query OK, 1 row affected (0.01 sec)

mysql> show tables;
+--------------+
| Tables_in_db |
+--------------+
| sbux_users   |
+--------------+
1 rows in set (0.00 sec)

mysql> SELECT * FROM sbux_users;
+----+---------+--------+
| id | name    | desc   |
+----+---------+--------+
|  1 | zhangs  | goods1 |
|  2 | zhangs2 | goods2 |
+----+---------+--------+
2 rows in set (0.00 sec)
```

#### 2.从库验证

```mysql
mysql> use db;
mysql> show tables;
+--------------+
| Tables_in_db |
+--------------+
| sbux_users   |
+--------------+
1 rows in set (0.00 sec)

mysql> SELECT * FROM sbux_users;
+----+---------+--------+
| id | name    | desc   |
+----+---------+--------+
|  1 | zhangs  | goods1 |
|  2 | zhangs2 | goods2 |
+----+---------+--------+
2 rows in set (0.00 sec)
```

## mysql半同步配置

### 主库配置
#### 1.配置半同步



```mysql
# 安装主库的半同步插件
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
# 打开半同步
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
Query OK, 0 rows affected (0.00 sec)
# 查看半同步相关参数Rpl_semi_sync_master_status为ON，半同步打开成功
mysql> show status like 'Rpl%';
+--------------------------------------------+-------+
| Variable_name                              | Value |
+--------------------------------------------+-------+
| Rpl_semi_sync_master_clients               | 1     |
| Rpl_semi_sync_master_net_avg_wait_time     | 0     |
| Rpl_semi_sync_master_net_wait_time         | 0     |
| Rpl_semi_sync_master_net_waits             | 12    |
| Rpl_semi_sync_master_no_times              | 0     |
| Rpl_semi_sync_master_no_tx                 | 0     |
| Rpl_semi_sync_master_status                | ON    |
| Rpl_semi_sync_master_timefunc_failures     | 0     |
| Rpl_semi_sync_master_tx_avg_wait_time      | 746   |
| Rpl_semi_sync_master_tx_wait_time          | 8953  |
| Rpl_semi_sync_master_tx_waits              | 12    |
| Rpl_semi_sync_master_wait_pos_backtraverse | 0     |
| Rpl_semi_sync_master_wait_sessions         | 0     |
| Rpl_semi_sync_master_yes_tx                | 12    |
+--------------------------------------------+-------+
14 rows in set (0.00 sec)
```

#### 2.新增配置项

/var/workspace/mysql_3306/data/my.cnf配置文件新增项，数据库启动后启用半同步

```bash
# 半同步插件加载
plugin-load=rpl_semi_sync_master=semisync_master.so
# 打开主的半同步
rpl_semi_sync_master_enabled=1
```

### 从库配置

#### 1.配置半同步

```mysql
# 安装从库的半同步插件
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
# 打开半同步
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
# 重启从的同步线程
mysql> STOP SLAVE IO_THREAD;
mysql> START SLAVE IO_THREAD;
# 查看半同步相关参数Rpl_semi_sync_slave_status为ON，半同步打开成功
mysql> show status like 'Rpl%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Rpl_semi_sync_slave_status | ON    |
+----------------------------+-------+
1 row in set (0.00 sec)
```

#### 2.新增配置项

/var/workspace/mysql_3306/data/my.cnf配置文件新增项，数据库启动后启用半同步

```bash
# 从库半同步插件加载
plugin-load=rpl_semi_sync_slave=semisync_slave.so
# 打开从的半同步
rpl_semi_sync_slave_enabled=1
```

### 半同步验证

同 [数据验证](#数据验证)

