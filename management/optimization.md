# 查询优化

本文简述查询优化相关的知识点.

## 使用索引

参考[index](./index.md)

## MySQL 查询优化程序

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

## 选择利于高效查询的数据类型

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

## 选择利于高效查询的表存储格式

对于 MySQL V5.7, Mariadb V10.3.1 以上已经不适用, 默认采用 Barracuda 格式.
