## 表信息查看

#### 查看表状态

```
mysql> show table status like 't_user%';
+--------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
| Name         | Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time | Check_time | Collation       | Checksum | Create_options | Comment |
+--------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
| t_user       | InnoDB |      10 | Dynamic    |    2 |           8192 |       16384 |               0 |            0 |         0 |              7 | 2020-09-06 22:46:31 | NULL        | NULL       | utf8_bin        |     NULL |                | ???     |
| t_user_roles | InnoDB |      10 | Dynamic    |    3 |           5461 |       16384 |               0 |        16384 |         0 |             14 | 2020-09-06 22:46:31 | NULL        | NULL       | utf8_bin        |     NULL |                |         |
+--------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+-------------+------------+-----------------+----------+----------------+---------+
```

#### 查看数据占用空间MB

```
mysql> select table_schema, sum(data_length+index_length)/1024/1024 as total_mb, \
sum(data_length)/1024/1024 as data_mb, sum(index_length)/1024/1024 as index_mb, \
count(*) as tables, curdate() as today from information_schema.tables group by table_schema order by 2 desc;

+--------------------+------------+------------+------------+--------+------------+
| table_schema       | total_mb   | data_mb    | index_mb   | tables | today      |
+--------------------+------------+------------+------------+--------+------------+
| mysql              | 7.89616680 | 7.68229961 | 0.21386719 |     31 | 2020-09-26 |
| information_schema | 0.15625000 | 0.15625000 | 0.00000000 |     61 | 2020-09-26 |
| sc_auth            | 0.09375000 | 0.09375000 | 0.00000000 |      6 | 2020-09-26 |
| wechat_spider      | 0.03125000 | 0.01562500 | 0.01562500 |      1 | 2020-09-26 |
| sys                | 0.01562500 | 0.01562500 | 0.00000000 |    101 | 2020-09-26 |
| performance_schema | 0.00000000 | 0.00000000 | 0.00000000 |     87 | 2020-09-26 |
+--------------------+------------+------------+------------+--------+------------+
14 rows in set, 2 warnings (0.71 sec)
```

#### 查看数据表量较大的表

```
mysql> SELECT 
TABLE_SCHEMA AS database_name,
TABLE_NAME AS table_name,
TABLE_ROWS AS table_rows,
ENGINE AS table_engine,
ROUND((DATA_LENGTH)/1024.0/1024, 2) AS Data_MB,
ROUND((INDEX_LENGTH)/1024.0/1024, 2) AS Index_MB,
ROUND((DATA_LENGTH+INDEX_LENGTH)/1024.0/1024, 2) AS Total_MB,
ROUND((DATA_FREE)/1024.0/1024, 2) AS Free_MB
FROM information_schema.`TABLES` AS T1
WHERE T1.`TABLE_SCHEMA` NOT IN('performance_schema','mysql','information_schema')
ORDER BY T1.`TABLE_ROWS` DESC;
+---------------+--------------------------+------------+--------------+---------+----------+----------+---------+
| database_name | table_name               | table_rows | table_engine | Data_MB | Index_MB | Total_MB | Free_MB |
+---------------+--------------------------+------------+--------------+---------+----------+----------+---------+
| finalschedule | gc_schedule_scheduler    |       1596 | MyISAM       |    0.05 |     0.02 |     0.07 |    0.00 |
| opensips      | version                  |         58 | InnoDB       |    0.02 |     0.00 |     0.02 |    0.00 |
| sbux_cac      | t_comm_field             |         42 | InnoDB       |    0.02 |     0.02 |     0.03 |    0.00 |
| sbux_cac      | t_asset_channel_auth_dtl |         37 | InnoDB       |    0.02 |     0.00 |     0.02 |    0.00 |
+---------------+--------------------------+------------+--------------+---------+----------+----------+---------+
10 rows in set, 2 warnings (0.54 sec)
```

#### 查看碎片较多的表

```
SELECT 
TABLE_SCHEMA AS database_name,
TABLE_NAME AS table_name,
TABLE_ROWS AS table_rows,
ENGINE AS table_engine,
ROUND((DATA_LENGTH)/1024.0/1024, 2) AS Data_MB,
ROUND((INDEX_LENGTH)/1024.0/1024, 2) AS Index_MB,
ROUND((DATA_LENGTH+INDEX_LENGTH)/1024.0/1024, 2) AS Total_MB,
ROUND((DATA_FREE)/1024.0/1024, 2) AS Free_MB,
ROUND(ROUND((DATA_FREE)/1024.0/1024, 2) /ROUND((DATA_LENGTH+INDEX_LENGTH)/1024.0/1024, 2)*100,2)AS Free_Percent
FROM information_schema.`TABLES` AS T1
WHERE T1.`TABLE_SCHEMA` NOT IN('performance_schema','mysql','information_schema')
AND ROUND(ROUND((DATA_FREE)/1024.0/1024, 2) /ROUND((DATA_LENGTH+INDEX_LENGTH)/1024.0/1024, 2)*100,2) >10
AND ROUND((DATA_FREE)/1024.0/1024, 2)>100
ORDER BY ROUND(ROUND((DATA_FREE)/1024.0/1024, 2) /ROUND((DATA_LENGTH+INDEX_LENGTH)/1024.0/1024, 2)*100,2) DESC;
```

#### 查看某库下所有表的大小

```
mysql> select table_name, (data_length/1024/1024) as data_mb , (index_length/1024/1024) \
as index_mb, ((data_length+index_length)/1024/1024) as all_mb, table_rows \
from information_schema.tables where table_schema = 'sbux_cac';
+------------------------------+------------+------------+------------+------------+
| table_name                   | data_mb    | index_mb   | all_mb     | table_rows |
+------------------------------+------------+------------+------------+------------+
| t_api_def                    | 0.01562500 | 0.01562500 | 0.03125000 |          7 |
| t_api_doc                    | 0.04687500 | 0.01562500 | 0.06250000 |          4 |
| t_api_permission             | 0.01562500 | 0.01562500 | 0.03125000 |         15 |
| t_asset_account_inf          | 0.01562500 | 0.00000000 | 0.01562500 |          0 |
+------------------------------+------------+------------+------------+------------+
35 rows in set (0.10 sec)
```



