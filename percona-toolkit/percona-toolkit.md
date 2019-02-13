# Percona-toolkit

percona toolkit是一个有30多个mysql工具的工具箱。兼容mysql,percona server,mariadb；它可以帮助DBA自动化的管理数据库和系统任务。

## 安装

先到 [percona](https://www.percona.com/downloads/percona-toolkit/LATEST/) 下载 rpm 文件.

``` shell
## 安装依赖
yum install perl-IO-Socket-SSL perl-DBD-MySQL perl-Time-HiRes perl-TermReadKey perl-IO-Socket-SSL -y
## 安装下载的 rpm 文件
rpm -ivh /path/to/percona-toolkit-3.0.12-1.el7.x86_64.rpm
```

## 工具集

各工具的[在线文档](https://www.percona.com/doc/percona-toolkit/LATEST/index.html)

### pt-archive

数据库归档工具，查询源数据，导出数据到文件\表\不做操作，然后删除源数据。小心使用，这个工具在归档数据以后默认会把源数据删除，所以需要手工加上-no-delete，我少有归档需求，而且偶尔用不熟练的话，风险也较大。

### pt-duplicate-key-checker

用来检查mysql表是否有重复的索引和外键。这个工具审查show create table的结果，如果发现不同索引包含相同的列，或者一个覆盖索引最左边的类和其他索引一样，它会打印出可疑的索引，默认只会针对同一种类型的索引做比较。它也可以用来查找重复的外键，一个重复的外键必须是在同一个表中和其他外键包含相同的列，而且参考的父表也相同。

### pt-mysql-summary

返回MySQL的状态和配置统计信息。

### pt-config-diff

比较MySQL配置文件的不同之处。

### pt-online-schema-change

在线变更数据库表结构，工具的工作原理是先创建一张和源表相同的空表，然后修改它的表结构。然后从源表拷贝数据过来，当所有数据拷贝过来以后，删除源表，将新表改名为源表。在变更过程中，任何对源表的改变都会通过触发器，应用到新表，所以源表默认不能有触发器。

### pt-query-digest

是用于分析mysql慢查询的一个工具，它可以分析binlog、General log、slowlog，也可以通过SHOWPROCESSLIST或者通过tcpdump抓取的MySQL协议数据来进行分析。可以把分析结果输出到文件中，分析过程是先对查询语句的条件进行参数化，然后对参数化以后的查询进行分组统计，统计出各查询的执行时间、次数、占比等，可以借助分析结果找出问题进行优化。

### pt-stalk

用来收集mysql的信息，通常数据库如果有一些突发的性能问题，这种问题又不是随时出现的话，就可以用到### pt-stalk了，他可以监控数据库，然后设定一个阀值，超过阀值，就记录数据库和系统相关的数据，以便分析。

### pt-summary

查看系统的相关统计信息。

### pt-diskstats

查看系统磁盘IO相关信息。

### pt-table-checksum

通过在master执行checkum检查复制的一致性，如果slave产生不一致的结果，那么证明slave和master数据不一致。使用DSN参数指定master的连接方式，如果任何不一致的数据被发现的话，EXIT STATUS状态都是非0的，或者产生了一些warnings或者错误。

这个工具专注于查找出不同的数据，如果你需要解决数据不同步的问题，可以使用### pt-table-sync。

### pt-table-sync

同步mysql服务器之间的数据。这个工具会改变数据，为了最大程度的安全，你需要在使用之前备份你的数据，如果同步一个复制的slave服务器使用–replicate或者–sync-to=master方法，通常是通过改变master，而不是直接改变slave，这是通常最安全的方法来完成主从的一致性。改变slave是解决问题的根源，但是通过改变master的数据，不会对master产生影响，实际上只会影响slave。

### pt-show-grants

列出MySQL所有用户及权限。

### pt-slave-delay

检查从库复制延迟。
