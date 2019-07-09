# USER

## CREATE

```sql
-- 使用新的密码认证方式
create user 'exporter'@'%' identified by '123' with max_user_connections 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
flush privileges;
```

``` sql
CREATE USER 'MY_USER'@'localhost' IDENTIFIED WITH mysql_native_password;
--- 使用旧密码格式加密, 已不推荐。这是为了解决部分旧客户端用旧的密码认证方式访问高版本的服务器。
SET old_passwords = 0;
SET PASSWORD FOR 'MY_USER'@'localhost' = 'MY_PASS';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'My_USER'@'localhost';
GRANT USAGE ON *.* TO 'MY_USER'@'localhost' WITH MAX_USER_CONNECTIONS 3;
flush privileges;
```

### secure_auth 与 old_passwords

secure_auth 为了防止低版本的MySQL客户端(<4.1)使用旧的密码认证方式访问高版本的服务器
MySQL 5.6.7开始secure_auth 默认为启用值1。但也可用skip-secure-auth禁用。

```sql
select version() , @@global.secure_auth;
```

* 如果old_passwords=1，则old_password()和password()加密结果相同都是16位。
* 如果old_passwords=0，则old_password()加密结果是16位。password()加密结果是41位.

```sql
select @@session.old_passwords,  @@global.old_passwords ,
 old_password('123'), password('123')\G
-- ***************************[ 1. row ]***************************
-- @@session.old_passwords | 0
-- @@global.old_passwords  | 0
-- old_password('123')     | 773359240eb9a1d9
-- password('123')         | *23AE809DDACAF96AF0FD78ED04B6A265E05AA257
```

### 查看授权

``` sql
-- 第一种方法
show grants for user_name;

-- 第二种
-- 全局授权
select * from mysql.user where user='user_name'\G;
-- 数据库级授权
select * from mysql.db where user='user_name'\G;
-- 表级授权
select * from mysql.tables_priv where user='user_name'\G;
-- 列级授权
select * from mysql.columns_priv where user='user_name'\G;

```

## DROP

``` sql
-- 用户 XXX 下面没有任何对象；这样才可以使用这个命令，否则就会报错；
drop user 'XXX';

-- 用户 XXX 下面有对象，要用级联方式
drop user 'XXX' cascade;
```
