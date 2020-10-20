# 优化

本文简述优化相关的知识点. 优化是基于测试的. 没有明确目标的优化都是不可取的.

## 系统

* 打开文件限制
* Core 文件限制
* swappiness

```shell
# 打开文件限制
# vi /etc/security/limits.conf
# 需重启机器
mysql soft nofile 65535
mysql hard nofile 65535

# Core 文件限制
mysql soft core unlimited
mysql hard core unlimited

# 重启后查看
ulimit -Sn  # 打开文件限制: soft
ulimit -Hn  # 打开文件限制: hard
ulimit -Sc  # core文件限制: soft
ulimit -Hc  # core文件限制: hard

# 虚拟内存控制
sysctl -w vm.swappiness=1
```

## 配置

```conf
# 非交互式连接的最大闲置时间
wait_timeout=1800
# 交互式连接的最大闲置时间
interactive_timeout=1800

# 最大连接数, 根据实际情况调整
max_connections=151
# 禁用 DNS. 要求授权表和连接中必须使用 IP 或 localhost.
skip_name_resolve

## innodb
# innodb_buffer_pool_size , innodb_buffer_pool_instances
# 每个pool实例的大小: innodb_buffer_pool_size / innodb_buffer_pool_instances
# 应保证每个实例至少 1GB

# 内存缓冲池总大小. 如果是机器为 mariadb 单独使用, 则可以控制在上限为物理内存的80%. 物理内存越大比例可以越高. 和连接数有关系.
# 可以写 MB, GB, TB, PB
innodb_buffer_pool_size=4096MB
innodb_buffer_pool_instances=2

## redo log 大小
## 增大意味着更少的磁盘 I/O, 但是, 崩溃后恢复时间更长.
## 设置参考: 比较粗糙的: innodb_log_file_size = innodb_buffer_pool_size / 4 / innodb_log_files_in_group
# innodb log group 中每个 Redo 日志文件大小. Redo log 用于服务崩溃重启后恢复
innodb_log_file_size=512M
# 每组 redo log 文件数
innodb_log_files_in_group=2
# redo log 写缓冲区
innodb_log_buffer_size=16MB

# 设置多少个事件后同步 binlog 到磁盘. 范围: 0~4294967295. 0: 由系统 flush binlog 文件到磁盘. 1: 每事务就 flush file.
sync_binlog=1
# innodb flush 日志的时机.
# 0: commit 后啥也不做. 每秒写 redo log 缓存, 刷盘, 性能最好. 服务崩溃丢失最后1秒数据.
# 1: 每事务即写 redo log, 又主动刷盘, 最慢.
# 2: 先写 redo log 缓存, 然后再每秒主动刷盘. 服务崩溃, 无影响, 系统崩溃丢失未刷盘数据.
# 3: 模拟group commit
innodb_flush_log_at_trx_commit=1
# 启用双写. 可以解决部分写失效问题, 但会带来更多的写负载, 大概会降低 10% 左右
innodb_doublewrite=1

# group commit for binlog
# 加大, 优点: 提高事务吞吐量. 缺点: 事务延迟返回. 慎重!!
# 延迟 binlog 刷盘, 等待binlog刷盘的个数. 默认为0(即不启用). 等待时间不超过 binlog_commit_wait_usec 
binlog_commit_wait_count.
# 延迟 binlog 刷盘, 等待的时间, 单位: 微秒. 在 binlog_commit_wait_count = 0时无效.
binlog_commit_wait_usec

# 每张表是否独立表空间文件. 默认已经是启用.
innodb_file_per_table=1
# innodb 打开表的最大数量.
innodb_open_files=800


# 访问 INFORMATION_SCHEMA.TABLES or  INFORMATION_SCHEMA.STATISTICS tables, 或者运行 metadata statements such as SHOW INDEX or SHOW TABLE STATUS 时更新统计信息. 建议关掉.
innodb_stats_on_metadata=0

# 在一个表缓存实例中最大保持打开的表的数量.
table_open_cache=2000

# 查询缓存大小. 优先级高于 query_cache_type. 弊大于利, 建议关掉. Mysql8.0 都删除了这个功能.
query_cache_size=0
# 查询缓存类型. 同样的查询语句在缓存命中时直接返回结果. 在表内数据不经常改动时启用会提升查询速度. 但是, 如果经常改动会给服务器带来负担, 降低性能.
query_cache_type=0


```

## 内存

### 理论内存

[在线计算工具](http://www.mysqlcalculator.com/)

```python
used_memory_size = key_buffer_size + query_cache_size + tmp_table_size + innodb_buffer_pool_size + innodb_additional_mem_pool_size + innodb_log_buffer_size
+ max_connections * (
sort_buffer_size + read_buffer_size + read_rnd_buffer_size + join_buffer_size + thread_stack + binlog_cache_size
)
```

## 查询优化

### 使用索引

参考[index](./index.md)

### MySQL 查询优化程序

* 使用 EXPLAIN 分析语句
  * 基本用法: 在 mysql 交互客户端中输入: EXPLAIN statement
  * Mariadb 可以使用 ANALYZE statement, ANALYZE 会先调用优化器(optimizer) 执行语句, 然后输出分析结果.
* 比较相同类型的列值. 比较值/被比较值的类型完全一致时, 查询性能最高. 完全一致是指: INT/INT, 或 CHAR(10)/CHAR(10), 如果比较值/被比较值类型分别为: INT/BIGINT, 或 CHAR(10)/CHAR(12) 都被认为类型不一致.
* 让索引列在比较表达式中单独出现. 避免在函数或算术表达式中使用索引, 会造成[索引失效](./index.md#索引失效).
  * where mycol * 2 < 4 (索引失效); where mycol < 4/2 (索引有效)
  * where YEAR(date_col) < 1990 (索引失效); where date_col < '1990-01-01' (索引有效)
* 不要在 LIKE 模式的开始位置使用通配符(索引失效). REGEXP 表达式也绝不会被优化.
* 测试查询的各种替代形式, 并多次运行它们. 优化程序在许多场合对连接的优化效果**好于**对子查询的优化效果.
* 避免过多使用自动类型转换.
* 使用 mysql 的性能跟踪功能, 默认是关闭的
  * 开启 profiling : set [ global ] profiling=1.
  * 执行 SQL 语句.
  * 查看总体性能统计信息: show profiles. 它会显示 query_id, duration, query(语句)
  * 查看某条语句具体的统计信息: show profile [ type [, type ] ... ] for query < query_id > [ limit *row_count* ]. type 包括: all, block io, context switches, cpu, ipc, memory, page faults, source, swaps

### 选择利于高效查询的数据类型

* 多用数字运算, 少用字符串运算
* 当较小类型够用时, 就不用较大类型
* 把数据列声明成 NOT NULL
* 考虑使用 ENUM 列
* 使用 PROCEDURE ANALYSE(), 让 MySQL 给出各列的优化数据类型的建议.
  * SELECT * FROM tbl_name PROCEDURE ANALYSE(); 给出表里各列的优化数据类型的建议
  * SELECT * FROM tbl_name PROCEDURE ANALYSE(16, 256); 告知PROCEDURE ANALYSE() 不要建议这样的 ENUM 类型: 它包含的值超过 16 个 或者 总长度超过 256 个字节. 如果不加限制, 那么输出结果会很长, 对于 ENUM 类型的定义, 可读性很差.
* 整理表碎片. 修改频繁的表, 往往会产生大量的碎片. 碎片造成磁盘空间利用率低, 查询性能下降.
  * 适用于各种存储引擎的碎片整理方法: 先用 mysqldump 转储表, 然后再用转储文件重建它: mysqldump db_name tbl_name > dump.sql; mysql db_name < dump.sql;
* 把数据压缩到 BLOB 或 TEXT 列. 使用 BLOB 或 TEXT 列存储那些可以在应用程序里压缩和解压缩的数据(在数据库里将这些数据看做一个整体).
* 使用合成索引. 一种做法是使用单独列存储某些难以索引的数据的 HASH 值. HASH 值非常适用精确查找, 不适用范围搜索. 数字型散列值的存储效率非常高.
* 避免检索很大的 BLOB 或 TEXT 值. 采用合成索引是比较好的选择.
* 把 BLOB 或 TEXT 列剥离出来形成一个单独的表. 这样可以减少主表里的碎片. 当在主表上 SELECT * 时, 避免拉取大的 BLOB 或 TEXT 值.

### 选择利于高效查询的表存储格式

对于 MySQL V5.7, Mariadb V10.3.1 以上已经不适用, 默认采用 Barracuda 格式.
