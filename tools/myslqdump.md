# myslqdump

## 常用参数

- --all-databases, -A: 导出全部数据库
- --all-tablespaces, -Y: 导出全部表空间
- --no-tablespaces, -y: 不导出任何表空间信息
- --add-drop-database: 每个数据库创建之前添加 drop database 语句
- --add-drop-table: 每个数据表创建之前添加 drop table 语句. 默认: 打开. 使用--skip-add-drop-table取消选项.
- --add-locks: 加锁每条 INSERT 语句. 默认: 打开.  使用--skip-add-locks取消选项.
- --complete-insert, -c: 使用完整的 INSERT 语句(包含列名). 可以提高效率, 但是可能受到 max_allowed_packet 参数的影响导致插入失败
- --compress, -C : 在 client/server 间传输数据时启用压缩
- --database, -B: 导出指定名称的多个数据库. 参数后面的名称都作为数据库名.
- --flush-logs, -F: 开始导出每个数据库前 flush logs.
- --flush-privileges: 在导出 mysql 数据库之后, 执行一条 FLUSH PRIVILEGES 语句. 该选项应该用于导出mysql数据库和依赖mysql数据库数据的任何时候。
- --force, -f: 忽略导出过程中的 SQL 错误.
- --ignore-table: 不导出指定的表. 每次指定一个表.
- --no-create-db, -n: 不导出 CREATE DATABASE 语句.
- --no-create-info, -t: 不导出 CREATE TABLE 语句.
- --no-data, -d: 不导出数据.
- --single-transaction: 仅在存储引擎支持 MVCC 的情况下工作, 当前仅 Innodb 有效. 该选项会关闭 --lock-tables 选项.
- --replace: 使用 REPLACE INTO 代替 INSERT INTO.
- --hex-blob: 使用十六进制格式导出二进制字段. 如果有二进制数据必须使用该选项.
- --master-data=#: 将 binlog 的位置和文件名追加到输出文件中. 如果为1，将会输出CHANGE MASTER 命令；如果为2，输出的CHANGE  MASTER命令前添加注释信息。该选项将打开--lock-all-tables 选项，除非--single-transaction也被指定（在这种情况下，全局读锁在开始导出时获得很短的时间；其他内容参考下面的--single-transaction选项）。该选项自动关闭--lock-tables选项。

## 注意问题

- 在加入 --flush-logs 参数时, mysqld 需要有权限写 log-error 指定的文件. 否则, 不报错, 但是无法正确导出.
