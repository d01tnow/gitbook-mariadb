# Galera 集群

Galera 集群相关的知识点

## 集群信息

### grastate.dat

识别最新节点ID是通过对比不同节点上GTID的值来进行，grastate.dat文件中保留了这些信息：在此文件中可以发现当前节点最新状态ID的值，当节点正常退出Galera集群时，会将GTID的值更新到该文件中。
依次停止Galera不同节点上数据服务，会发现后来服务停止服务节点上最后提交事务序号seqno越大，这对于查找数据最新的节点非常有帮助。

``` shell
[root@controller01 bin]# systemctl stop mariadb
[root@controller01 bin]# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    93d771bf-1f55-11e7-ac0b-86a123048de2
seqno:   1132817
cert_index:
[root@controller01 bin]# ssh controller02 systemctl stop mariadb
[root@controller01 bin]# ssh controller02 cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    93d771bf-1f55-11e7-ac0b-86a123048de2
seqno:   1132946
cert_index:
[root@controller01 bin]# ssh controller03 systemctl stop mariadb
[root@controller01 bin]# ssh controller03 cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    93d771bf-1f55-11e7-ac0b-86a123048de2
seqno:   1132971
cert_index:
```

如果该节点正在运行服务，grastate.dat文件中最后事务提交序号seqno为-1

``` shell
cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    93d771bf-1f55-11e7-ac0b-86a123048de2
seqno:   -1
cert_index:
```

最后提交事务ID不停再变化，可以通过以下命令查询：SHOW STATUS LIKE 'wsrep_last_committed';

### gvwstate.dat

当集群形成或改变Primary Component时，节点会创建或更新gvwstate.dat文件，确保节点保留最新Primary Component的状态，如果节点无法连接，可以利用该文件进行参照，如果节点正常关闭，该文件会被删除。

文件有两部分组成：
本节点信息：my_uuid统一标识符。
Primary Component的相关信息：包含在#vwbeg和#vwend之间。
view_id 由view_type、view_uuid和view_seq三部分组成的标识符：
view_type 规定用3表示Primary信息。
view_uuid 和view_seq形成惟一的标识符。
bootstrap 显示该节点是否启动，但这不影响Primary Component的恢复流程。
member 显示primary component中节点的UUID。

``` shell
cat /var/lib/mysql/gvwstate.dat
my_uuid: ccebf42b-2319-11e7-a371-0285ad3982af
#vwbeg
view_id: 3 4fe6aa21-23d6-11e7-9627-7e448dd4f8cd 7
bootstrap: 0
member: 4fe6aa21-23d6-11e7-9627-7e448dd4f8cd 0
member: ccebf42b-2319-11e7-a371-0285ad3982af 0
member: d9200df6-2319-11e7-a505-5ef957851eb2 0
#vwend
```
