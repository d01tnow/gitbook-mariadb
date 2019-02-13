# Configuration

  mariadb的配置项说明, 以及 Garela 集群配置项说明

## 配置文件位置

默认的配置文件名为 my.cnf. 它可以存在多个位置, 按固定层级加载, 最后加载的覆盖前面加载的相同字段的配置, 扩充没有的字段.
当 SYSCONFDIR 没有定义时
Location | Scope
---- | ----
/etc/my.cnf | Global
/etc/mysql/my.cnf | Global
$MYSQL_HOME/my.cnf | Server
defaults-extra-file | 由命令参数 --defaults-extra-file=path 指定
~/.my.cnf | User

如果 SYSCONFDIR 定义时
Location | Scope
---- | ----
$SYSCONFDIR/my.cnf | Global
$MYSQL_HOME/my.cnf | Server
defaults-extra-file | 由命令参数 --defaults-extra-file=path 指定
~/.my.cnf | User

### Docker镜像中的配置

Mariadb 镜像的数据存储文件是在容器的 /var/lib/mysql 目录下，配置文件 /etc/mysql/my.cnf 又包括在 /etc/mysql/conf.d 目录中找到的任何文件.cnf。此目录中的文件中的设置将扩充或覆盖中的设置 /etc/mysql/my.cnf。

### 配置文件语法

* 注释用 #
* 忽略空行
* 分组方式组织, \[grou-name\]
* !include filename 用于包含其他配置文件
* !include directory 用于包含其他目录下的配置文件(扩展名.cnf, .ini), 按字母顺序加载
* loose做前缀的配置项是可选项, 表示配置项不被支持时不会产生错误. 比如: loose-innodb_file_per_table111

可被识别的 group
Group | Description
---- | ----
client | 所有客户端读取的选项, 比如: mysqldump
client-server | mariadb client 和 mariadb server 读取的选项. 这是 mariadb 独有的.
client-mariadb | mariadb client 读取的选项
mysqld | mysqld server 读取, 同时支持 mysql, mariadb
mariadb | mariadb 的 mysqld 服务读取
mysqld-VERSION | 由指定版本的 mysqld 读取. 同时支持 mysql, mariadb
mariadb-VERSION | 由指定版本的 mariadb mysqld 服务读取
PROGRAMNAME | 由指定名称的程序读取. 比如: mysqldump

按照约定, 配置中的服务器变量名中可以使用 '-' 或 '_' 分隔字符串, 两者可以互换. 命令行中使用 '-' 分隔字符串.

## Mariadb配置

[具体参数描述参考][3]

``` conf
# mysql 配置文件例子

## 用于 client
[client]
# 默认连接字符集
default_character_set=utf8
# port, 默认 3306
port=3306

## 用于 mysql 程序
[mysql]
# 启用命令自动补全, 默认是关闭的: no_auto_rehash
auto_rehash
loose_abort_source_on_error

## 用于 mysql 或者 mariadb 的 mysqld 服务
[mysqld]

##------------网络相关------------------------------------

## 禁用 DNS. 要求授权表中必须使用 IP 或 localhost.
## 默认启用 DNS. 即注释掉 skip_name_resolve
skip_name_resolve

## 网络包大小. 建议 16M.
## 默认: 1 MB
max_allowed_packet=16M

## 服务端关闭交互式连接等待的空闲时间. 交互式连接即使用 mysql_real_connect() 函数中使用了 CLIENT_INTERACTIVE 选项. 比如: mysql 控制台客户端是交互式的, JDBC连接的客户端是非交互式的.
## 可通过show processlist输出中Sleep状态的时间
## 在连接启动时, 根据连接类型来确认会话的超时时间设置为 wait_timeout 还是 interactive_timeout .
## 默认: 28800 S
interactive_timeout=1800

## 服务端端关闭非交互式连接等待的空闲时间
## 默认: 28800 S(秒), 即 8 小时
wait_timeout=1800

## bind 地址, mysql 在 linux 上默认使用 unix socket 通讯. mariadb 默认绑定 0.0.0.0. 在 debian 和 ubuntu 上默认绑定 127.0.0.1
bind_address=0.0.0.0

## 监听端口
#port=3306

## listen 队列长度.
## 不设置或者=0, 或者使用命令行参数 --autoset-back-log 时,
## version >= 10.1.7 时, 默认值 min(900,(50 + max_connections/5);
## version > 10.0.8 && version<10.1.7 时,默认值min(150,max_connections)
## version <= 10.0.8 时, 默认值: 50.
#back_log=511

## 最大连接数
max_connections=400

##------------数据目录相关------------------------------------

## 数据存储目录. 如果数据文件不用绝对路径, 那么都会保存在该路径下
datadir=/var/lib/mysql
## 所有日志文件和 .pid 文件的 basename, 官方强烈推荐用这种方式设置日志和 pid 文件的文件名
## 如果不指定它, 那么日志文件的 basename 依赖 hostname, 这会有问题
log_basename=mysql

## 是否启用 binary logging, log_bin[=name], 写了即启用, =后面是bin log 的名字
## 默认文件名: $datadir/${log-basename}-bin 或者 $datadir/mysql-bin
## 官方推荐使用 log_basename 指定 basename, 不要依赖 hostname.
## 默认值: OFF. 如果未通过 --log-bin=path 方式指定文件名
log_bin

## 启用慢查询日志, 默认值: 0
slow_query_log=1
## 慢查询阈值. 单位: 秒. version >= 10.1.13 默认为 10.000000.
long_query_time=10


## 设置多少个事件后同步 binlog 到磁盘, 默认值: 0
## 0: 由操作系统处理刷盘
## 1: 每次写入就刷盘, 最安全但最慢模式
## 范围: 0 ~ 4294967295
#sync_binlog=0

## binary logging 格式. 默认值: version>=10.2.4: MIXED. <=10.2.3: STATEMENT
## STATEMENT：只记录修改数据的sql，可能会有问题
## ROW: 记录哪条数据被修改了，修改成什么样了，可能会有大量日志
## MIXED: 一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式
## Galera 集群要求必须为: ROW
binlog_format=ROW

## 从其他节点同步的数据是否也写日志. 默认: 0. 必须开启.
log_slave_updates=1

##--------------------------字符集相关------------------------------------
## 初始化连接时的参数
## 客户端比较和排序用的字符集
init_connect='SET collation_connection = utf8_general_ci'
## 客户端. java
init_connect='SET NAMES utf8'
## 忽略客户端设置的字符集, 都按照 init_connect 设置的
skip-character-set-client-handshake
## server 级别字符集, utf8mb4 是最多4字节的 utf8 编码, utf8 是最大3字节编码
character_set_server=utf8
## server 级别 collation 编码
collation_server=utf8_general_ci

##--------------------引擎相关------------------------------------------
## 默认存储引擎
## Galera 集群当前仅支持事务引擎 XtraDB/InnoDB, 不支持非事务引擎 MyISAM.
default_storage_engine=InnoDB
## 自增字段锁模式.
## 0: 每次申请会锁表; 1: 默认值, 会生成连续的自增值; 2: 不使用表级锁, 速度快, 但是基于语句(statement-based)的复制不安全
## Galera 集群要求必须为: 2 . 因为 Galera 集群不能使用表级锁.
innodb_autoinc_lock_mode=2
## 日志刷盘模式
## 0: 事务 commit 后不刷盘. 性能最好. 但服务崩溃时擦除最后一秒的事务. 考虑使用 Galera 集群, 使用 0 模式.
## 1: 默认值. 配合 sync-log=1 获得最大的容错级别. 每次事务 commit 都会把 log buffer 刷盘.
## 2: 每次 commit 后写入 log buffer, 每秒刷盘. 性能也不错, 但是有 OS崩溃或者掉电, 有可能丢失最后一秒内的事务数据.
## 3: (version>=10.0), 模拟 group commit (3 syncs per group commit)
innodb_flush_log_at_trx_commit=0
## 日志与数据刷盘方法
## O_DSYNC: 用于日志刷盘. 数据刷盘用 fsync()
## O_DIRECT: directio(), 用于打开数据文件, 数据和日志刷盘用 fsync()
## fsync: 在 unix 上的默认值.
## 其他的参考: <https://mariadb.com/kb/en/library/xtradbinnodb-server-system-variables/#innodb_flush_method>
innodb_flush_method=O_DIRECT

## innodb 缓冲池最大值
## TODO:
## 可以设置为物理内存的 60% ~ 80%. 如果超过 2 GB, 最好调整下 innodb-buffer-pool-instances
## 默认值: 134217728 (128MB). 可以使用 M, G 等后缀.
innodb_buffer_pool_size=128M
## 缓冲池实例数量. 每个实例管理的池的大小为: innodb_buffer_pool_size/innodb_buffer_pool_instances
## 默认值: 1
## 设置参考: 每个实例的大小(innodb_buffer_pool_size/innodb_buffer_pool_instances): [1GB, 2GB]
## 动态设置时要确保:  innodb_buffer_pool_size / innodb_buffer_pool_chunk_size 不能大于 1000
## [设置参考1](https://mariadb.com/kb/en/library/innodb-system-variables/#innodb_buffer_pool_size)
## [设置参考2](https://mariadb.com/kb/en/library/setting-innodb-buffer-pool-size-dynamically/)
innodb_buffer_pool_instances=1

## innodb log group 中每个 Redo 日志文件大小. Redo log 用于服务崩溃重启后恢复.
## v10.0 之前, innodb_log_file_size * innodb_log_files_in_group 不能超过 4GB, v10.0 之后不能超过 512G
## innodb_log_files_in_group 默认 2
## 增大意味着更少的磁盘 I/O, 但是, 崩溃后恢复时间更长.
## 设置参考: 比较粗糙的: innodb_log_file_size = innodb_buffer_pool_size / 4 / innodb_log_files_in_group
innodb_log_file_size=48M
innodb_log_files_in_group=2

## innodb 日志缓冲区大小
## 默认: 16MB
## 增大可以减少磁盘 I/O
## 可通过查看innodb_log_waits状态，如果不为0的话，则需要增加innodb_log_buffer_size
innodb_log_buffer_size=16M

## innodb 表数据文件分配方式.
## 默认: 0. 表和索引信息存储在系统空间内.
## 1: 或 ON. 每个表和索引信息保存在自己 .ibd 文件中. 这样可以更快的完成类似 'TRUNCATE' 操作. 当删除或截断一个表时, 可以回收未使用的空间. 这样配置的另一个好处是可以将某些表放在其他存储设备上, 提升磁盘 I/O 负载
innodb_file_per_table=1

## innodb 打开表的数量. 过小会影响 recovery 的效率.
## 默认值: 300
innodb_open_files=800

## 是否启用 innodb 双写. 可以解决部分写失效问题, 但会带来更多的写负载, 大概会降低 10% 左右
## 默认: 0
## InnoDB使用了一种叫做doublewrite的特殊文件flush技术，在把pages写到date files之前，InnoDB先把它们写到一个叫doublewrite buffer的连续区域内，在写doublewrite buffer完成后，InnoDB才会把pages写到data file的适当的位置。如果在写page的过程中发生意外崩溃，InnoDB在稍后的恢复过程中在doublewrite buffer中找到完好的page副本用于恢复
innodb_doublewrite=1

## 访问 INFORMATION_SCHEMA.TABLES or  INFORMATION_SCHEMA.STATISTICS tables, 或者运行 metadata statements such as SHOW INDEX or SHOW TABLE STATUS 时更新统计信息
## mysql 在 v5.6.6 以后已经默认是 OFF 了. mysqltuner 工具也建议把这个选项关掉, 据说会严重影响服务器性能.
## 默认: 1 or ON
innodb_stats_on_metadata=0
##-----------------缓存----------------

## 在表进行order by和group by排序操作时，由于排序的字段没有索引，会出现Using filesort，为了提高性能，可用此参数增加每个线程分配的缓冲区大小。一般出现Using filesort的时候，要通过增加索引来解决。
## 默认: 256KB，这个参数不要设置过大，一般在128～256KB即可。
sort_buffer_size=256K

## 表随机读取时每个线程分配的缓存区大小. 非索引字段排序时, 会利用该缓存区. 不宜过大.
## 默认: 256KB.
read_rnd_buffer_size=512K

## 在一个表缓存实例中最大保持打开的表的数量.
## 默认: V10.1.7 之前是 400, 之后 2000
table_open_cache=2000

## 查询缓存类型. 同样的查询语句在缓存命中时直接返回结果. 在表内数据不经常改动时启用会提升查询速度. 但是, 如果经常改动会给服务器带来负担, 降低性能. 弊大于利, 建议关掉. Mysql8.0 都删除了这个功能.
## 默认: 0 或 OFF.
## 1 或 ON: SELECT 语句的结果被缓存, 除非指定了 SQL_NO_CACHE.
## 2 或 DEMAND: 带 SQL_CACHE 子句的查询语句被缓存.
query_cache_type=0
## 查询缓存大小. 即使 query_cache_type=0, query_cache_size>0 , 也会分配该内存.
## 在 query_cache_size=0 时, 即使 query_cache_type != 0 , 也会禁用查询缓存
query_cache_size=0

## 临时表(不是指 MEMORY 表)占用的最大内存大小, 设置 = max_heap_table_size
## 默认值: 16MB
## 通常复杂的 GROUP BY 查询导致创建的临时表大小超过这个限制, 只能在磁盘上创建临时表, 从而降低了性能
## 查看在磁盘上创建临时表的比例, 如果高就适当增加该值, 同时修改 max_heap_table_size.
## show status like 'created_tmp%'
## created_tmp_disk_tables: 表示在磁盘上创建的临时表数量
## created_tmp_tables: 表示创建临时表总量
tmp_table_size=16M
max_heap_table_size=16M


```

## Garela配置

``` conf
[galera]
## 是否启用 wsrep 复制
## 默认: 0 or OFF
wsrep_on=1
## wsrep 库的位置
wsrep_provider="/usr/lib/galera/libgalera_smm.so"
## wsrep 库选项. 根据停机时间决定参数大小，选项中还有很多参数，暂用不上
wsrep_provider_options="gcache.size=300M; gcache.page_size=300M"

## 状态快照传输中使用的身份验证信息. 格式: user:password
## 在 wsrep_sst_method=rsync 时无效
#wsrep_sst_auth="root:123"
## 状态快照传输方法
## 默认: rsync
## mysqldump: 阻塞式的方法, 最慢的方法
## rsync: 此选项比 mysqldump 传大型数据集快得多。但是,只能在服务器启动时采用，数据接受服务器需要同数据共享服务器Donor配置相同（例如，服务器间innodb_file_per_table配置必须完全一致）。
#wsrep_sst_method="rsync"

## TODO:
## 集群地址
wsrep_cluster_address="gcomm://maria_boot,maria_node1,maria_node2"
## 集群名称
wsrep_cluster_name="maria_cluster"
## 节点名称, 集群内唯一
## 默认: 服务器主机名
wsrep_node_name="maria_node2"
## 节点地址. 格式: ip[:port]
## 默认: 第一个网络接口的地址, 比如: etho0. 如果没有则为 0.0.0.0. 默认端口: 4567
wsrep_node_address=maria_node2
## 用于并行从属写入集时使用的线程数. 但是请注意，如果一致性问题经常发生，将值设置为1可能会修复问题。
## 默认: 1
## 设置参考: 设置为 CPU 数的 2 倍
wsrep_slave_threads=2

```

[1]: <https://mariadb.com/kb/en/library/configuring-mariadb-with-mycnf/> "mariadb configuration"
[2]: <https://my.oschina.net/xsh1208/blog/1052781> "utf8 与 utf8mb4 区别"
[3]: <https://mariadb.com/kb/en/library/server-system-variables>
