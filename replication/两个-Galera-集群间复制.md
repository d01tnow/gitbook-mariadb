# 两个 Galera 集群间复制

以下基于 mariadb 10.3 验证. 参考[官方文档](https://mariadb.com/kb/en/configuring-mariadb-replication-between-two-mariadb-galera-clusters/).

前置条件:

1. 主控机上安装 ansible 环境, Mariadb-client.
2. 被控主机中安装 mariabackup 工具. [安装方法](../tools/installing-tools.md).
3. 被控主机 sudo 权限 或者 mysql user(容器内 mysql 用户:组 -- mysql:mysql . uid:gid -- 999:999).
4. 数据库 root 或者 有 REPLICATION SLAVE 权限的 user.
5. 第一个集群节点到第二个集群节点免密访问.

以下例子中集群 1 (3个节点) 简称 A, ip 为 192.168.150.24~3, 集群 2 (3个节点) 简称 B, ip 为 192.168.150.21~3 . 端口都是默认的 3306. A 为 master, B 为 slave. 

为了模拟生产环境, 初始状态下, A 启动, B 未启动. 

## 集群安装

### ansible方式

解压 [test_ansible.zip](./test_ansible.zip). 修改文件中的 ip 指向正确的主机. 

说明: 

1. mariadb 集群对应主从模式或主主模式中的 A 集群. mariadb_repl 集群对应主从模式或主主模式中的 B 集群.
2. 集群最小节点数量为 2. 比如: mariadb_repl .

相关命令参见 playbooks/mariadb_readme.md 和 playbooks/mariadb_repl_readme.md.

#### 安装步骤

``` shell
# init, 输入 sudo 用户密码
ansible-playbook -i ../inventory pb_mariadb.yml -t init,updateconf -K
# deploy
ansible-playbook -i ../inventory pb_mariadb.yml -t deploy
# 启动服务, 检查端口
ansible-playbook -i ../inventory pb_mariadb.yml -t start,check
```



## 主从模式

要想在两个 Galera 集群间复制, A, B 都必需的配置项.

``` ini
# A,B 的所有节点都要配置
#  从其他节点同步的数据也写日志
log_slave_update=ON

# server_id 用于区分 master 和 slave. 
# A 的所有节点相同. B 的所有节点相同. 但是, A 的 server_id 一定不等于 B 的 server_id
# 当前使用版本mariadb 10.3.4. N 取值 [1, 2**32) .  
server_id=N

# ** 所有节点的 gtid_domain_id 都要不同 ** 
# 并且, ** gtid_domain_id != wsrep_gtid_domain_id **
# M 取值范围: [0, 2**32)
gtid_domain_id=M

# binlog 存放路径. 同一集群内相同.
# 可以省略路径. 此时, 需要配置 datadir (数据存放目录) 和 log_basename(binlog, slowlog, errorlog, pid 等文件的基础文件名). 
log_bin[=name]

# 数据文件的基础文件名
log_basename=mariadb

# 开启 wsrep_gtid_mode
wsrep_gtid_mode=ON
# wsrep 的 gtid_domain_id
# **同一个集群内相同. 不同集群一定要不同.**
# N 取值范围 [0, 2**32)
wsrep_gtid_domain_id=N

```

当前生产环境中, A 已经配置了 log_slave_update=ON, server_id=1, log_bin=/var/log/mysql/mysql-bin 

### 设置第一个集群

1. 使用命令方式

``` shell
# 通过 mariadb client - mysql 设置
# A1~3
mysql -u root -p -h 192.168.150.24 << EOF
set global wsrep_gtid_mode=ON;
set global wsrep_gtid_domain_id=1;
set global gtid_domain_id=50001;
EOF
mysql -u root -p -h ipA2 << EOF
set global wsrep_gtid_mode=ON;
set global wsrep_gtid_domain_id=1;
set global gtid_domain_id=50002;
EOF
mysql -u root -p -h ipA3 << EOF
set global wsrep_gtid_mode=ON;
set global wsrep_gtid_domain_id=1;
set global gtid_domain_id=50003;
EOF

# 创建备份用户
mysql -u root -p -h 192.168.150.24 << EOF
CREATE USER 'repl'@'192.168.%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.*  TO 'repl'@'192.168.%';
FLUSH PRIVILEGES;
EOF
```

2. 使用ansible脚本方式

``` shell
ansible-playbook pb_mariadb_config_wsrep_gtid.yml -t set_gtid -e set_gtid_enabled=1 -e mdb_grp=mariadb
```

pb_mariadb_config_wsrep_gtid.yml

``` yaml
- hosts: localhost
  gather_facts: no
  tasks:
  - block:
    - name: 打印组信息
      debug:
        msg: "{{item}}"
      loop: "{{ groups[mdb_grp]|sort }}"
    - name: 打印 mariadb_gtid_domain_id
      debug: var=hostvars[item].mariadb_gtid_domain_id
      loop: "{{ groups[mdb_grp]|sort }}"
    - name: 打印 mariadb_wsrep_gtid_domain_id
      debug: var=hostvars[item].mariadb_wsrep_gtid_domain_id
      loop: "{{ groups[mdb_grp]|sort }}"
    tags: set_gtid


  - name: 确认组信息
    pause: minutes=2
    tags: set_gtid

  - name: configuring wsrep gtid mode
    shell: |
      mysql -h {{hostvars[item].ansible_ssh_host}} -p{{hostvars[item].mariadb_root_password}} -u root << EOF
      -- 打开 gtid 模式
      set global wsrep_gtid_mode=ON;
      -- 设置写集的 gtid domain id. 一个集群内所有节点相同.
      set global wsrep_gtid_domain_id={{hostvars[item].mariadb_wsrep_gtid_domain_id}};
      -- 设置 domain 内的 gtid. 要保证, 一个集群内所有节点 **唯一**.
      set global gtid_domain_id={{hostvars[item].mariadb_gtid_domain_id}}

      EOF
    loop: "{{ groups[mdb_grp]|sort }}"
    when: set_gtid_enabled | bool
    tags: set_gtid

  - name: check wsrep gtid
    shell: |
      mysql -h {{hostvars[item].ansible_ssh_host}} -p{{hostvars[item].mariadb_root_password}} -u root -e "show global variables like '%gtid_%';"
    loop: "{{ groups[mdb_grp]|sort }}"
    register: check_result
    tags:
    - set_gtid
    - check_gtid
  - name: 打印检查结果
    debug: var=check_result.results|map(attribute='stdout_lines')|list
    tags:
    - set_gtid
    - check_gtid


```

### 备份第一个集群数据

``` shell
# 登录 A1
# 进入 mariadb 容器
# 建立备份目录
mkdir -p /var/lib/mysql/backup/fullbackup
# 备份数据.
# --target-dir 指明备份数据存放目录
mariabackup --backup \
	--target-dir=/var/lib/mysql/backup/fullbackup \
	--user=root --password=root_pwd
	
# 准备
mariabackup --prepare --target-dir=/backup/fullbackup

# 退出容器
# 拷贝准备好的备份数据到 B1. 需要先做 A1 到 B1 的 ssh 免密. 并且 /backup 有读写权限
# /Madb/mariadb/backup 是 /backup 在主机上的映射
sudo chown -R user /Madb/mariadb/backup/fullbackup

# 进入 B 机器, 建立 /backup 目录
ssh user@192.168.150.21 
sudo mkdir /backup
sudo chown user /backup
# CTRL +D 退出 B, 回到 A

rsync -avrP /Madb/mariadb/backup/fullbackup 192.168.150.21:/backup
```

### 部署第二个集群

假设第二个集群部署在 /app/midserv/mariadb_repl 目录. 数据目录为 /Madb/mariadb_repl/data. 

**注意**: 因为集群 B 使用的是集群 A 全量备份恢复的数据库. 所以, 配置中 root 用户的密码要同集群 A 的相同(例子中的 wsrep_sst_auth="root:the_same_as_A".

``` shell
# 登录 B1
# 拷贝备份数据到数据目录
# /backup 需要读权限
# /Madb/mariadb_repl/data 需要写权限
mariabackup --copy-back  --datadir=/Madb/mariadb_repl/data \
	--target-dir=/backup
	
# 修改数据目录权限. 因容器中 mysql:mysql 的 uid:gid 为 999:999
sudo chown -R 999:999 /Madb/mariadb_repl/data

# 检查 B1 的配置, 被控机B1上 /app/midserv/mariadb_repl/config/cluster.cnf
server_id = 200
wsrep_gtid_mode=ON
wsrep_gtid_domain_id=200
gtid_domain_id=50201
wsrep_sst_auth="root:the_same_with_A"

# 检查 B2 的配置, 被控机B2上 /app/midserv/mariadb_repl/config/cluster.cnf
server_id = 200
wsrep_gtid_mode=ON
# 同 server_id
wsrep_gtid_domain_id=200
# 算法: 50000 + server_id + 在集群中的编号
gtid_domain_id=50202
wsrep_sst_auth="root:the_same_with_A"

# 检查 B3 的配置, 被控机B3上 /app/midserv/mariadb_repl/config/cluster.cnf
server_id = 200
wsrep_gtid_mode=ON
# 同 server_id
wsrep_gtid_domain_id=200
# 算法: 50000 + server_id + 在集群中的编号
gtid_domain_id=50203
wsrep_sst_auth="root:the_same_with_A"
```

### 启动第二个集群的 boot 节点

``` shell
# 带 --wsrep-new-cluster 参数启动集群
# 查看第一个集群的 master 状态
mysql -u root -p -h 192.168.150.24 -e 'show master status;'
# 假设输出
# File	Position	Binlog_Do_DB	Binlog_Ignore_DB
# mysql-bin.000018	2418

# 设置第二个集群的 B1 的 master 为 A1
# 并检查结果
mysql -u root -p -h 192.168.150.21 << EOF
change master to master_host="192.168.150.24", master_user="repl", master_password="password", master_log_file="mysql-bin.000018", master_log_pos=2418;
start slave;
show slave status\G;
EOF

# 结果中出现以下,表示正常(出现Slave_IO_Running: Connecting 时，需要等几秒后再次查询)
# Slave_IO_Running               | Yes
# Slave_SQL_Running              | Yes
# Seconds_Behind_Master 表示同步时间延后 master 的秒数. 0 表示同步.
# Seconds_Behind_Master          | 0
```

### 启动第二个集群的其他节点

``` shell
# 启动后检查集群状态
mysql -u root -p -h ipB2 -e "show status like 'wsrep%';"
# wsrep_cluster_size           | 3
# wsrep_local_state            | 4                                                                
# wsrep_local_state_comment    | Synced

# 设置第二个集群所有节点为只读
mysql -u root -p -h 192.168.150.21 -e 'set global read_only=on;'

mysql -u root -p -h ipB2 -e 'set global read_only=on;'

mysql -u root -p -h ipB3 -e 'set global read_only=on;'
```

## 主主模式

在主从模式的基础上进行设置, 形成主主模式.

``` shell
# 查看第二个集群(B)的 master 状态
mysql -u root -p -h 192.168.150.21 -e 'show master status;'
# 假设输出
# File	Position	Binlog_Do_DB	Binlog_Ignore_DB
# mysql-bin.000004	96

# 设置第一个集群的 A1 的  master 为 B1
# 并检查结果
mysql -u root -p -h 192.168.150.24 << EOF
change master to master_host="192.168.150.21", master_user="repl", master_password="password", master_log_file="mysql-bin.000004", master_log_pos=96;
start slave;
show slave status\G;
EOF

# 结果中出现以下,表示正常(出现Slave_IO_Running: Connecting 时，需要等几秒后再次查询)
# Slave_IO_Running               | Yes
# Slave_SQL_Running              | Yes
# Seconds_Behind_Master 表示同步时间延后 master 的秒数. 0 表示同步.
# Seconds_Behind_Master          | 0
```







