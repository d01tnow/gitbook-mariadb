# Installing tools

## 安装

### CentOS7 

[参考](https://mariadb.com/kb/en/yum/#installing-the-most-common-packages-with-yum)

#### 准备 repo

1. 安装指定版本

``` shell
# 先安装 repo 文件
sudo vi /etc/yum.repo.d/mariadb.repo

[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

# 保存退出

sudo yum clean all && sudo yum makecache
# 安装 GPG Public Key
sudo rpm --import https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
```



1. 安装最新版本

``` shell
# 安装 repo 文件
curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
sudo yum clean all && sudo yum makecache
# 安装 GPG Public Key
sudo rpm --import https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
```

#### 安装服务或工具

``` shell
# 10.4
sudo yum install MariaDB-server galera-4 MariaDB-client MariaDB-shared MariaDB-backup MariaDB-common

# 10.3
sudo yum install MariaDB-server galera MariaDB-client MariaDB-shared MariaDB-backup MariaDB-common
```

