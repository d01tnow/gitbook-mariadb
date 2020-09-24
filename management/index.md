# 使用SQL管理索引

## mysql支持的索引类型

支持的索引类型由存储引擎决定.

索引特性 | InnoDB | MyISAM | MEMORY
---- | ---- | ---- | ----
是否允许使用 NULL 值 | 是 | 是 | 是
每个索引的列数 | 16 | 16 | 16
每个表的索引数 | 64 | 64 | 64
索引行的最大长度(字节) | 3072 | 1000 | 3072
是否支持列前缀索引 | 是 | 是 | 是
列前缀的最大长度(字节) | 767 | 1000 | 3072
是否支持 BLOB/TEXT 索引 | 是 | 是 | 否
是否支持 FULLTEXT 索引 | ver >= 5.6.4 | 是 | 否
是否支持 SPATIAL 索引 | 否 | 是 | 否
是否支持 HASH 索引 | 否 | 否 | 是

mysql 支持的索引类型:

* 主键索引(PRIMARY INDEX): 是聚集索引, 即表数据的主键顺序决定了物理顺序, 物理顺序同主键顺序一致. 如果表没有主键, mysql 会选择第一个 UNIQUE 索引（所有列都是 NOT NULL）的作为聚集索引。 如果没有这样的索引，mysql 创建一个对用户不可见的自增ID列作为聚集索引.
* 唯一索引 (UNIQUE): 对于单列索引, 不允许出现重复值. 对于多列(复合)索引, 不允许出现重复的组合值. 更新操作需要将数据页读入内存, 判断是否唯一, 唯一才能更新.
* 常规(非唯一性)索引: 它可以让你获得索引的好处, 但是会出现重复值的情况. 更新记录 change buffer(改善写操作, 使用 buffer pool 中的内存, 也会被持久化) 中就返回了.
* FULLTEXT 索引: 它可以用于全文检索.
* SPATIAL 索引: 它只适用于包含空间值的 MyISAM 表.
* MEMORY 索引: 它是 MEMORY 表的默认索引类型, 不过可以通过创建 BTREE 索引来改写它.

## 显示索引

``` SQL
SHOW INDEX FROM tbl_name \G
```

其中 Cardinality 会有如下的含义：

1. 列值代表的是此列中存储的唯一值的个数（如果此列为primary key 则值为记录的行数）
2. 列值只是个估计值，并不准确。默认的InnoDB存储引擎对随机取8个叶子节点的 Leaf Page (统计每个页不同记录的个数)进行采样。叶子节点数 > 8 时, 每次得到的Cardinality值可能不同的.
3. 列值不会自动更新，需要通过Analyze table来更新一张表或者mysqlcheck -Aa来进行更新整个数据库。
4. 列值的大小影响Join时是否选用这个Index的判断。
5. 创建Index时，MyISAM的表Cardinality的值为null，InnoDB的表Cardinality的值大概为行数。
6. MyISAM与InnoDB对于Cardinality的计算方式不同。

## 创建索引

通过 ALTER TABLE 语句比 CREATE INDEX 语句创建索引更灵活, 因为它可以创建 MySQL 所支持的任何一种索引.

* ALTER TABLE tbl_name ADD INDEX       index_name (index_columns);
* ALTER TABLE tbl_name ADD UNIQUE      index_name (index_columns);
* ALTER TABLE tbl_name ADD PRIMARY KEY index_name (index_columns);
* ALTER TABLE tbl_name ADD FULLTEXT    index_name (index_columns);
* ALTER TABLE tbl_name ADD SPATIAL     index_name (index_columns);

其中 tbl_name 是表名, index_name 是索引名, 可选, 不写时取第一个索引列的名字. index_columns 是要索引的列(多个列用 ',' 分隔, 又称复合索引). 如果要建前缀索引, 可以在列名后加括号, 括号里填写前缀长度. 例如: index_column(number). 对于列的值较长，比如BLOB、TEXT、VARCHAR，就必须建立前缀索引.
如果指明 PRIMARY KEY 或 SPATIAL , 则它必须是 NOT NULL 的. 其他类型索引允许包含 NULL 值.
PRIMARY KEY 和 UNIQUE 都可以限制某个索引**只包含唯一值**. 它们的区别是:

* 每个表只能包含一个 PRIMARY KEY. 因为 PRIMARY KEY 索引的名字总是 PRIMARY, 而同一个表不允许有同名索引. 但可以有多个 UNIQUE 索引.
* PRIMARY KEY 不能包含 NULL 值, 而 UNIQUE 索引可以. 如果某个 UNIQUE 索引包含了 NULL 值, 那么它可以包含多个 NULL 值. 这是因为, NULL 值不与任何值相等, 甚至和另一个 NULL 值也不相等.

如果想在 CREATE TABLE 同时创建索引, 那么它的语法和 ALTER TABLE 很相似, 只不过在定义各列的基础上增加一些索引创建子句.

``` SQL
CREATE TABLE tbl_name
(
  -- 列定义

  -- 索引定义
  INDEX       index_name (index_columns),
  UNIQUE      index_name (index_columns),
  PRIMARY KEY index_name (index_columns),
  FULLTEXT    index_name (index_columns),
  SPATIAL     index_name (index_columns),
)
```

作为一种特殊情况, 通过在某列的定义末尾加上 PRIMARY KEY 或 UNIQUE 子句, 可以创建一个单列的 PRIMARY KEY 或 UNIQUE 索引.

在某些场合, 只能对列前缀(而不是整个列)进行索引.

* 对于 BLOB 或 TEXT 列, 只能创建前缀型索引.
* 索引行的长度等于构成该索引的各列的索引部分的长度总和. 如果总长度超过了索引行所能容纳的最大字节数, 那么可以通过索引列的前缀来 "缩短" 这个索引.

FULLTEXT 索引里的列是以 **满列值方式** 进行索引的, 不能进行前缀索引. 即使指定前缀长度, 也会被忽略.

## 删除索引

DROP INDEX index_name ON tbl_name
DROP INDEX `PRIMARY` ON tbl_name

如果不知道索引名称, 则可以使用 SHOW INDEX FROM tbl_name 或者 SHOW CREATE TABLE tbl_name 查询.
当删除列时, 相应的索引也会隐式地删除对应的列. 没有列的索引就被删除了.

## 索引失效

* 查询条件中对索引列进行运算, 运算包括( +, -, *, /, DIV(整除), %(取模), like '%_'(%_放在前面)).
* 条件中类型错误, 比如字段类型为 VARCHAR, where 中使用 NUMBER.
* 如果列类型是字符串，那一定要在条件中数据使用引号，否则不使用索引.
* 条件中对索引使用内部函数(如: WHERE DAY(column)), 这种情况应该建立基于函数的索引. 如select * from template t where ROUND(t.logicdb_id) = 1; 此时应该建ROUND(t.logicdb_id)为索引，mysql8.0开始支持函数索引，5.7可以通过虚拟列的方式来支持，之前只能新建一个ROUND(t.logicdb_id)列然后去维护.
* 如果条件有 OR ，只有在 OR 条件中的每个列都有索引时才有效, 否则无效.
* 组合索引遵循最左原则. 如: A,B,C 字段建立组合索引(A,B,C), 当 WHERE A='...' 时有效, 而当 WHERE B='...' 时无效.
* IS NULL, IS NOT NULL，!= 是否走索引，也是根据成本判断的。[参考](https://www.cnblogs.com/niuben/p/11197945.html)
* 重复数据较多的列, 即使建立索引效果也不好. 引擎判断全表查询花费更低时, 也会使索引失效.

## 挑选索引

•回表：先查辅助索引找到主键，再根据主键找到数据。

•最左前缀原则：索引匹配过程采用的原则。可以是复合索引的最左 N 个字段，页可以是字符串索引的最左 M 个字符。

•覆盖索引：指 select 的字段已经包含在索引中，不需要再读取数据行。主键一定是被索引覆盖的。

•索引下推：在索引遍历过程中，过滤掉索引中包含的字段不满足条件的记录，减少回表次数。

主要参考"MySql技术内幕 第5版 5.1.3"

* 为用于搜索, 排序或分组的列创建索引, 而对用作输出显示的列则不用创建索引. 也就是说那些出现在 WHERE 子句, 连接子句, ORDER BY, GROUP BY 子句中的列创建索引. 而只出现在 SELECT 后面的输出列不用建立索引.
* 认真考虑数据列的基数. 列的基数是指它所容纳的所有非重复值的个数. 列数据中包含的唯一值越多, 重复值越少, 索引的效果越好
* 索引短小值. 应该尽量选用较小的数据类型. 短小值可以让操作更快; 可以让索引短小, 减少磁盘 I/O; 可以在内存内缓存更多的索引. 对于 InnoDB, 它使用聚簇索引(clustered index, 把数据行和主键值存储在一起), 其他的索引都是二级索引(即把主键值和二级索引存储在一起). 在二级索引里进行查找, 会先得到主键值, 然后通过它定位到相应的行. 这意味着, 主键值在每个二级索引中都会重复出现. 因此, 如果主键值较长, 导致每个二级索引都需要占用更多的存储空间.
* 索引字符串值的前缀. 想要对字符串列进行索引, 应当尽可能指定前缀长度. 如果大多数值的前10个或20个字符都是唯一的, 那么就可以不用为整个列进行索引, 而只为前面的 10 个或 20 个字符进行索引.
* 利用最左前缀. 当创建包含 N 个列的复合索引时, 实际上会创建 n 个专供 MySql 使用的索引. 复合索引相当于多个索引. 如果通过调整顺序可以减少维护一个索引，那么顺序是优先考虑的。
* 不要建立过多索引. 索引占用磁盘空间, 影响写入操作的性能, 更新数据时很可能需要对索引进行重组.
* 让参与比较的索引类型保持匹配. InnoDB 总会使用 B+ 树索引. MyISAM 也会使用 B+ 树索引, 但对空间类型则会改用 R 树索引. MEMORY 存储引擎默认使用 HASH 索引, 但它也支持 B+ 树索引, 并允许你在两者间进行选择. 对于散列索引, 在使用 = 或 <=> 完成精确匹配的操作里, 速度非常快. 但在那些需要范围比较的操作里表现欠佳. 对于 B+ 树索引, 在使用 <, <=, =, >, >=, <>, != 和 BETWEEN 运算符, 进行精确比较或范围比较时, 比较高效.
* 利用慢查询日志找出那些性能低劣的查询, 进行优化.

### Explain

```php

10种常见的 type 
NULL>system>const>eq_ref>ref>ref_or_null>index_merge>range>index>ALL
```

[参考](https://learnku.com/articles/38719)

## 实现细节的影响

* InnoDB 存储引擎表数据和索引保存在**同一个表空间**文件里.在默认情况下使用**一个**表空间, 这个表空间内管理着所有 InnoDB 表的数据存储和索引存储. 可以配置 InnoDB 引擎, 每个表都有自己的表空间.
* MyISAM 存储引擎则是把数据和索引分别保存在**不同**文件里. 一个表可以有多个索引, 但它们都保存在同一个索引文件里.

### 索引的代价

* 对于 InnoDB 系统表空间里的所有表, 都共享同一个存储空间池, 添加索引会使表空间里用于存储的空间减少的更快. 不过, 与 MyISAM 表所用的文件不同, InnoDB 系统表空间不会受到操作系统文件大小的限制, 因为可以配置它使用多个文件. 使用独立表空间的 InnoDB 表, 会把数据和索引存储在同一个文件里, 因此增加索引将导致表的大小更加快速地达到文件的最大限制大小.
* 对于 MyISAM 表, 大量的对它进行索引, 有可能导致索引文件比数据文件更快的到达文件最大限制大小.

## 参考

1. [Mysql使用自增ID主键与UUID的比较](https://blog.csdn.net/mchdba/article/details/52279523)
2. [自增ID,UUID还是雪花算法](https://blog.csdn.net/mchdba/article/details/52279523)
