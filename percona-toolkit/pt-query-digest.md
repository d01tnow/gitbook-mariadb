# Pt-query-digest

pt-query-digest是用于分析mysql慢查询的一个工具，它可以分析binlog、General log、slowlog，也可以通过SHOWPROCESSLIST或者通过tcpdump抓取的MySQL协议数据来进行分析。可以把分析结果输出到文件中，分析过程是先对查询语句的条件进行参数化，然后对参数化以后的查询进行分组统计，统计出各查询的执行时间、次数、占比等，可以借助分析结果找出问题进行优化。

安装参见[percona-toolkit](./percona-toolkit.md)

## 语法及选项

``` shell
pt-query-digest --help
Usage: pt-query-digest [OPTIONS] [FILES] [DSN]
--host      #指定MySQL地址;
--port      #指定MySQL端口;
--socket    #指定MySQL SOCK文件;
--user      #指定MySQL用户;
--password  #指定MySQL密码;
--type      #指定将要分析的类型,默认是slowlog,还有如binlog,general log等;
--charset   #指定字符集;
--filter    #对输入的慢查询按指定的字符串进行匹配过滤后再进行分析;
--limit     #限制输出结果百分比或数量,默认值是20,即将最慢的20条语句输出,如果是50%则按总响应时间占比从大到小排序,输出到总和达到50%位置截止;
--review    #将分析结果保存到表中,这个分析只是对查询条件进行参数化,一个类型的查询一条记录,比较简单;当下次使用--review时,如果存在相同的语句分析,就不会记录到数据表中;
--history   #将分析结果保存到表中,分析结果比较详细,下次再使用--history时,如果存在相同的语句,且查询所在的时间区间和历史表中的不同,则会记录到数据表中,可以通过查询同--CHECKSUM来比较某类型查询的历史变化;
--since     #从什么时间开始分析,值为字符串,可以是指定的某个"yyyy-mm-dd [hh:mm:ss]"格式的时间点,也可以是简单的一个时间值:s(秒)、h(小时)、m(分钟)、d(天),如12h就表示从12小时前开始统计;
--until     #截止时间,配合--since可以分析一段时间内的慢查询;
--log       #指定输出的日志文件;
--output    #分析结果输出类型,值可以是report(标准分析报告)、slowlog(Mysql slow log)、json、json-anon;一般使用report,以便于阅读;
--create-review-table     #当使用--review参数把分析结果输出到表中时,如果没有表就自动创建;
--create-history-table    #当使用--history参数把分析结果输出到表中时,如果没有表就自动创建;
```

更多用例参考[使用pt-query-digest分析MySQL日志](http://www.ywnds.com/?p=8179)
