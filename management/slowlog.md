# SlowLog

本文简述慢查询日志的设置, 查看, 分析, 清除等知识和操纵方法.

## 设置

通过配置文件静态设置. 需要重启服务才生效.

``` conf
## 启用慢查询日志, 默认值: 0
slow_query_log=1
## 慢查询阈值. 单位: 秒. version >= 10.1.13 默认为 10.000000.
long_query_time=10

## 日志文件路径
## 通常设置 datadir 和 log_basename=mysql,
## 则 variable : slow_query_log_file=${datadir}/${log_basename}-slow.log

## 未使用索引的查询(优先于阈值 long_query_time )是否记录到慢查询日志, 默认: 关闭: 0
log_queries_not_using_indexes=0
```

通过管理命令动态设置. 立即生效, 重启服务时使用配置文件中的配置, 缺失配置时使用默认配置值.

``` mysql
## 打开(=1)或关闭(=0)慢日志服务
## 全局
set global slow_query_log=0

## 关闭慢日志服务后, 可以设置新的慢日志的路径
set global slow_query_log_file=/path/to/new/slow.log

## 设置慢查询日志的阈值, 单位秒, 浮点数, 最大6位小数
## 设置全局(set global) 相关变量.设置 session(set) 相关变量
set global long_query_time=1.000000
```

## 查看慢查询日志

在命令行中查看相关变量

``` mysql
## 查看全局相关(show global variables ). 查看 session (show variables ) 相关
show variables where variable_name like 'slow%log%' or variable_name like 'long_query%';
```

### 查看慢日志文件

``` shell

less /var/lib/mysql/mysql-slow.log

Time: 2017-05-26T02:21:45.263281Z ## 执行sql的时间点
User@Host: root[root] @  [210.73.xxx.xxx]  Id: 14032 ## 执行sql的 用户@主机 信息
Query_time: 2.080283  Lock_time: 0.000038 Rows_sent: 1  Rows_examined: 10221443 ## 执行所用时间、锁定时间、返回行数、扫描行数
SET timestamp=1495765305; ## 执行sql时unix时间戳(s)，* 1000 转成java中的时间戳(ms)
select count(*) as col_0_0_ from course_info courseinfo0_ where courseinfo0_.merchantId=100  limit 1; ## sql语句

```

### 分析慢日志的工具

[pt-query-digest](../percona-toolkit/pt-query-digest.md) 是一款分析日志(包括慢日志, binlog, tcpdump 抓的mysql通讯包等)的工具, 替代 mysqlsla.

### 分析慢日志的方法

总体思路是: 通过慢日志或慢日志工具找到具体的语句, 然后通过 EXPLAIN 分析语句或打开 profiling, 通过执行语句后 show profiles 或 show profile for query < id > 等展示语句执行的性能指标. 参考[optimization](./optimization.md).

## 清空慢查询日志

以下步骤中的命令都是在 mysql 交互命令行中使用. 由于都是动态修改, 不会存到配置文件中, 如果需要永久修改, 需要修改配置文件.

1. 关闭慢查询服务: mysql> set global slow_query_log=0
2. 查看慢查询服务相关变量: mysql> show variables like '%slow%'
3. 设置新的慢查询文件路径: mysql> set slow_query_log_file=/path/to/new-slow.log
4. 如果需要修改慢查询阈值: mysql> set global long_query_time=1.000000
5. 启动慢查询服务: mysql> set global slow_query_log=1
6. 查看慢查询相关变量: mysql> show variables where variable_name like 'slow%log%' or variable_name like 'long_query%';
7. 测试: mysql> select sleep(10) as a, 1 as b; exit;
8. 在命令行下查看新的慢查询日志文件: less /path/to/new-slow.log
9. 把旧的慢日志文件移动到其他目录下: mv /path/to/old-slow.log /path/to/mv-dest/old-slow.log
