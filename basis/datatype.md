# Datatype

本文涉及 Mysql 数据类型的相关知识

## 概述

### 数字类型

数字类型包括: 整数, 定点数, 浮点数和位值. 除 BIT 外, 其他数字类型都可以带正负号. 整数可以被定义成 UNSIGNED , 取值范围最小值为0, 最大值扩大. DECIMAL 也可以被定义为 UNSIGNED, 但是, 不会扩大取值范围, 而只会 "砍掉" 负数部分.
数字类型分为 3 大类:

* 精确值类型: 整数 和 DECIMAL
* 浮点类型: 单精度(FLOAT) 和 双精度(DOUBLE), 是近似值
* BIT类型: 用于存储位域值

类型名称 | 含义 | 占用空间(字节)
---- | ---- | ----
TINYINT[ (M) ] | 非常小的整数 | 1个字节
SMALLINT[ (M) ] | 小整数 | 2个字节
MEDIUMINT[ (M) ] | 中等大小整数 | 3个字节
INT[ (M) ] | 标准大小的整数 | 4个字节
BIGINT[ (M) ] | 大整数 | 8个字节
FLOAT[ (M, D) ] | 单精度 | 4个字节
DOUBLE[ (M, D) ] | 双精度 | 8个字节
DECIMAL( [M[, D] ) | 定点数 | 由有效位数(M), 数学精度(D)决定, 默认: M=10, D=0
BIT[ (M) ] | 位值 | 由位数(M)决定, 1 <= M <= 64, 默认: M=1

说明: 上表中 [] 表示可选, () 表示必选. 所有整数中的 M 仅表示显示宽度, 不会改变存储大小. 浮点数和 DECIMAL 中的 M 表示小数点前面的数字个数, D 表示小数点后面的数字个数. DECIMAL 类型对于小数点前后的每一侧, 每 9 位数字需要 4 个字节, 最后剩下的数字需要 1~4 个字节. 一个 BIT[ (M) ] 大约需要 (M+7)/8 个字节的存储空间.

### 字符串类型

下表中 [] 表示可选, () 表示必选. L 表示数据的真实长度(单位:字节). M 表示字段定义长度. W 表示列字符集里最"宽"字符所占的字节数据. BLOB 系列保存二进制串, TEXT 系列保存字符串.

类型名称 | 含义 | 占用空间(字节)
---- | ---- | ----
BINARY[ (M) ] | M 个**字节,定长**二进制串 | M
VARBINARY(M) | 最大长度为 M 个**字节,变长**二进制串 | L+1 或 L+2
CHAR[ (M) ] | M 个**字符,定长**字符串 | M * W
VARCHAR(M) | 最大长度为 M 个**字符,变长**字符串 | L+1 或 L+2
TINYBLOB, TINYTEXT | 最长 2**8 - 1个字节 | L+1
BLOB, TEXT | 最长 2**16 - 1 个字节 | L+2
MEDIUMBLOB, MEDIUMTEXT | 最长 2**24 - 1 个字节 | L+3
LONGBLOB, LONGTEXT | 最长 2**32 - 1 个字节 | L+4
ENUM('v1', 'v2'...) | 最大 65535 个成员的枚举值 | 1 或 2个
SET('v1', 'v2'...) | 最大 64 个成员 | 1 / 2 / 3 / 4 / 8

二进制串类型和非二进制串类型的对应关系

二进制串类型 | 非二进制串类型 | 说明
---- | ---- | ----
BINARY | CHAR | 定长, 即使为 NULL , 也占用 M * 单位字节(比如: 1个字符占2个字节, 单位字节为2)的空间
VARBINARY | VARCHAR | 变长, 长度+数据方式存储, 长度部分占用 1~2 字节
BLOB | TEXT | 变长, 长度+数据方式存储, 长度部分占用 1~4 字节

CHAR 和 VARCHAR 的区别
类型 | 描述
---- | ----
CHAR | 定长. 存入时小于 M 个字符, 用空格补齐; 取出时, 移除尾部空格. 启用 PAD_CHAR_TO_FULL_LENGTH 时, 保留尾部空格
VARCHAR | 不定长. 存入取出时不会处理尾部空格.

VARCHAR(M) 的 M *理论*范围 1~65535, 但它实际能容量的最大字符个数肯定小于 65535. 这是因为在 MySQL 里, 行的最大长度为 65535 个字节. 下面是一些潜在的影响点.

* 一个 VARCHAR 长度 > 255 字节的列需要 2 个字节来存放字符串长度, 最终长度不能超过行的总长(65535字节).
* 1个字符使用 n 个字节表示, 那么由于受限于行的总长, 则最多的字符个数为 65535 / n.
* 同一行的其他列也会使 VARCHAR 列的最大长度减少

BINARY 和 VARBINARY 的区别
类型 | 描述
---- | ----
BINARY | 定长. 存入时源数据长度小于 M 字节, 以 0 补齐到 M 个字节; 取出时, 不处理.
VARBINARY | 不定长. 存入取出时不会处理数据.

BLOB 是二进制大对象类型. 使用字节为单位. TEXT 是非二进制大对象类型. 也使用字节为单位.
BLOB 和 TEXT 能否被索引, 取决于存储引擎. InnoDB 和 MyISAM 都支持 BLOB 和 TEXT 的索引. 但是必须指定前缀长度. TEXT 列的 FULLTEXT 索引以完整内容为基础, 忽略指定的前缀长度. Memory 表不支持 BLOB 和 TEXT 索引.

对于 BLOB 和 TEXT 需要注意以下几点.

* 多次执行删除或修改操作之后, 容易产生大量碎片. 如果是 MyISAM 引擎, 定期执行 OPTIMIZE TABLE 命令可以有效减少碎片, 改善系统性能.
* max_sort_length (默认值: 1024)系统变量会对 BLOB 和 TEXT 类型值的比较和排序产生影响.
* 对于数据量非常大的值, 可能需要调整 max_allowed_packet 参数的值. 对于所有想要使用大数据量的客户端程序, 则需要在客户端增大数据包的大小. 客户端程序 mysql 和 mysqldump 都支持使用启动选项来设置这个值.

## 如何选择数据类型

1. 值要表示为字符数据还是二进制数据 ? 非二进制串类型 : 二进制串类型.
2. 比较操作需要区分大小写吗 ? 非二进制串类型 : 二进制串类型. 字符集和排序规则是相关联的.
3. 想要少用存储空间吗 ? 可变长的类型 : 定长类型. 一种情况除外, 当所有数据长度都是定长的, 那么选择定长类型.
4. 列的取值是从有限的值里选取吗 ? ENUM 或 SET : 其他.
5. 可以修改尾部的填充吗 ? CHAR 或 BINARY : VARCHAR 或 VARBINARY, TEXT 或 BLOB

## 时态数据类型

类型名称 | 取值范围
---- | ----
DATE | '1000-01-01' ~ '9999-12-31'
TIME | '-838:59:59[ .000000 ]' ~ '838:59:59[ .000000 ]'
DATETIME | '1000-01-01 00:00:00[ .000000 ]' ~ '9999-12-31 23:59:59[ .999999 ]'
TIMESTAMP | '1970-01-01 00:00:00[ .000000 ]' ~ '2038-01-19 03:14:07[ .999999 ]'
YEAR | 1901 ~ 2155

如果要声明包含小数秒部分的时态类型列, 则定义为 cloumn_name type_name(fsp). 其中 type_name 为 TIME, DATETIME 或 TIMESTAMP, fsp 为取值 0 ~ 6 的整数, 表示小数秒精度, 默认为0.

MySQL 按照 UTC 时间存储 TIMESTAMP. 保存时, 服务会把它从会话时区转换为 UTC. 检索时, 从 UTC 转换为会话时区. SET time_zone='+08:00' 将会话时区调整为东八区(北京时间).

DATETIME 和 TIMESTAMP 自动特性

* 自动初始化: 如果在 INSERT 语句里**省略**了这两种类型的列, 那么列会被设置为当前时间戳.
* 自动更新: 对于已有行, 当把任何其他列更改为不同值时, 这两种类型的列都会被更新为当前时间戳(将列设置成它的当前值不算自动更新, 这种做法实际上是防止自动更新).

TIMESTAMP 列默认 NOT NULL, 即当用户显式的设置该列值为 NULL 时, 服务把该列值设置为当前时间戳. 如果定义时指定默认值为 NULL, 那么当用户显式的设置该列值为 NULL 时, 服务存储 NULL. 使用以下语法指定 TIMESTAMP 列.

``` SQL
 -- default_value 可以是: 0, CURRENT_TIMESTAMP 或 格式为 'YYYY-MM-DD hh:mm:ss' 的值
  col_name TIMESTAMP [ DEFAULT default_value ] [ ON UPDATE CURRENT_TIMESTAMP ]
```

DATETIME 列默认值为 NULL, 当用户显式的设置该列值为 NULL 时, 服务存储 NULL.

## 处理序列

MySQL 提供唯一编号的机制是使用 AUTO_INCREMENT 列属性.

### 通用的 AUTO_INCREMENT 属性

AUTO_INCREMENT 列必须按照以下条件进行定义.

* 每个表只能有一个列具有 AUTO_INCREMENT 属性, 并且它为整数数据类型(可以为浮点, 但是很少那样使用).
* 列必须建立索引. 最常见的情况是使用 PRIMARY KEY 或 UNIQUE 索引, 但也允许使用不唯一的索引.
* 列必须为 NOT NULL. 即使没有显式声明为 NOT NULL, MySQL 也会自动设置列为 NOT NULL.

AUTO_INCREMENT列具有以下行为.

* 不指定 AUTO_INCREMENT 列的值 或 指定 AUTO_INCREMENT 列值为 NULL , 将自动生成一个序列编号作为该列字段的值.
* 默认情况下, 指定 AUTO_INCREMENT 列值为 0 , 也会自动生成一个序列编号作为该列字段的值. 但如果启用了 NO_AUTO_VALUE_ON_ZERO 模式, 则将 0 作为列值.
* 要获得最近生成的序号, 可以调用 LAST_INSERT_ID() 函数. 这是会话相关函数. 如果当前会话里, 没生成过 AUTO_INCREMENT 的值, 那么 LAST_INSERT_ID() 返回0.
* 一次插入多行的 INSERT 语句, 将生成多个 AUTO_INCREMENT 的值, 但是 LAST_INSERT_ID() 仅返回第一个值.
* 如果使用 INSERT DELAYED, 那么要直到实际插入行时, 才会生成 AUTO_INCREMENT 的值, 那么不能依赖 LAST_INSERT_ID() 返回的序号值了.
* 对于某些存储引擎, 从序列顶端(从大值一端)删除的值可以被重用. 如果把表清空, 那么所有值都可以重用, 并且值从 1 开始.
* 如果使用 UPDATE 命令更新 AUTO_INCREMENT 列的值时, 如果值重复则报错; 否则, 如果值为 0, 那么列值被设置为 0 ( 不管是否启用 NO_AUTO_VALUE_ON_ZERO ). 如果值大于所有 AUTO_INCREMENT 列的值, 那么后续的新建行从这个新值+1开始.
* 如果根据 AUTO_INCREMENT 列的值 使用 REPLACE 更新行, 那么这个行的 AUTO_INCREMENT 值不变. 如果根据其他具有 PRIMARY_KEY 或 UNIQUE 索引的列的值, 使用 REPLACE 更新行, 且把 AUTO_INCREMENT 列值设为 NULL(或者 0), 而且没有启用 NO_AUTO_VALUE_ON_ZERO 时, 该列的值被更新为新的序号值.

### InnoDB 表的 AUTO_INCREMENT 列

* 可以在 CREATE TABLE 语句里, 使用 AUTO_INCREMENT = n 指定初值. 后续可以使用 ALTER TABLE 修改.
* 从序列顶端删除的值通常不能再重用. 但是, 通过 TRUNCATE TABLE 清空表, 那么序列被重置, 并重新从 1 开始. InnoDB 在内存里维护计数器, 它并未存储在表内部. 这意味着, 如果从这个序列的顶端删除了某些值, 然后重启服务, 那么这些值会被重用. 重启服务还将取消 CREATE TABLE 或 ALTER TABLE 语句设置的 AUTO_INCREMENT = n 选项.

### 在无 AUTO_INCREMENT 的情况下生成序列

MySQL 支持使用 LAST_INSERT_ID(expr) 函数生成序列编号的方法. 如果在插入或修改一个列时使用 LAST_INSERT_ID(expr), 那么下次不带参数的 LAST_INSERT_ID() 返回表达式 expr 的值. 而且, LAST_INSERT_ID() 是会话相关函数, 不受其他客户端影响.

``` SQL
-- 创建一个只有一行的表, 其中包含一个在每次都需要该序列里下一个值时都会进行更新的值.
CREATE TABLE seq_table (seq INT UNSIGNED NOT NULL);
INSERT INTO seq_table VALUES(0);
-- 使用 LAST_INSERT_ID(expr) 更新, 这种方法可以生成任意步长(甚至可以是负值)的序列编号
UPDATE seq_table SET seq = LAST_INSERT_ID(seq+100);
SELECT LAST_INSERT_ID();
```
