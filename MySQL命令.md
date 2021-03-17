<!-- TOC -->

- [命令行操作](#命令行操作)
- [权限与安全](#权限与安全)
- [临时开启通用查询日志](#临时开启通用查询日志)
- [sql_mode](#sql_mode)
- [二进制命令](#二进制命令)
- [复制、备份与恢复](#复制备份与恢复)
- [主从复制的服务器配置](#主从复制的服务器配置)
- [开发规范](#开发规范)
- [需要监控的参数](#需要监控的参数)
- [数据库预热](#数据库预热)
- [存储函数和函数](#存储函数和函数)
- [技巧](#技巧)
- [错误解决](#错误解决)
- [性能排查](#性能排查)

<!-- /TOC -->
# 命令行操作  
[新手MySQL工程师必备命令速查手册](https://mp.weixin.qq.com/s?__biz=MzkwOTIxNDQ3OA==&mid=2247533284&idx=1&sn=b97ed53cb2ab2bac1a82ed40e51a7d80&source=41#wechat_redirect)  
show status like 'Com_%';  
通过以上几个参数，可以很容易地了解当前数据库的应用是以插入更新为主还是以查询操作为主，以及各种类型的SQL 大致的执行比例是多少。  

- 查找数据库文件大小  
 ```
SELECT
	table_schema AS 'DATABASE',
	table_name AS 'TABLE',
	SUM( ROUND( ( ( data_length + index_length ) / 1024 / 1024 ), 2 ) ) 'Size IN MB '
FROM
	information_schema.TABLES 
WHERE
	information_schema.TABLES.TABLE_SCHEMA = 'boguan360' 
GROUP BY
	information_schema.TABLES.TABLE_SCHEMA,
	information_schema.TABLES.TABLE_NAME WITH ROLLUP  
  ```

ANALYZE TABLE tablename;  
check table tablename;  
optimize table tablename;  
这个命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费，但OPTIMIZE TABLE 命令只对MyISAM、BDB 和InnoDB 表起作用。  
  
 
mysqlbinlog --verbose master_bin.000001 (基于行的日志)  
SHOW BINLOG EVENTS IN 'mysql-bin.000001'  
myisampack  
mysqlslap  
[pt-query-digest](http://blog.csdn.net/seteor/article/details/24017913)  
show profile  
set profiling=1; show profiles;show profile for query quert_id;  
select state,sum(duration) as total_r,round(100*sum(duration)/(select sum(duration) from information_schema.profiling where query_id=?),2) as pct_r,count(*) as calls,sum(duration)/count(*) as "r/call" from information_schema.profiling where query_id=? group by state order by total_r desc  
  
mysqldumpslow -t10 /path/to/log3304/slowquery.log 访问时间最长的10个sql  
mysqldumpslow -s c -t10 /path/to/log3304/slowquery.log 访问次数最多的10个sql  
mysqldumpslow -s r -t10 /path/to/log3304/slowquery.log 访问记录集醉的的10个sql  
  
# 权限与安全  
- 权限授予  
'GRANT ALL PRIVILEGES ON `indu_program`.* TO \'credit_l\'@\'%\''  
'GRANT USAGE ON *.* TO \'credit_l\'@\'%\' IDENTIFIED BY PASSWORD \'*3100D4383269AAD781A4483B928CF9BDB8AC53C5\''  
- 权限移除  
GRANT ALL PRIVILEGES ON *.* TO 'cacti'@'%' IDENTIFIED BY 'cacti' WITH GRANT OPTION;    
- 重新载入赋权表    
FLUSH PRIVILEGES;    
- 收回权限(不包含赋权权限)    
REVOKE ALL PRIVILEGES ON *.* FROM cacti;    
REVOKE ALL PRIVILEGES ON cacti.* FROM cacti;    
- 收回赋权权限    
REVOKE GRANT OPTION ON *.* FROM cacti;    
- 重新载入赋权表    
FLUSH PRIVILEGES;    
  
  
#  临时开启通用查询日志  
set global general_log='on';  
explain extended  
show warnings  
perror 错误号  
show status like 'handler%';检查是否使用了索引  
  

  
# sql_mode  
解析  
这个sql_mode,简而言之就是：它定义了你MySQL应该支持的sql语法，对数据的校验等等。。  
- 如何查看当前数据库使用的sql_mode：  
select @@sql_mode;    
- sql_mode值的含义：  
ONLY_FULL_GROUP_BY：  
对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么将认为这个SQL是不合法的，因为列不在GROUP BY从句中  
STRICT_TRANS_TABLES：  
在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做任何限制  
NO_ZERO_IN_DATE：  
在严格模式，不接受月或日部分为0的日期。如果使用IGNORE选项，我们为类似的日期插入'0000-00-00'。在非严格模式，可以接受该日期，但会生成警告。  
NO_ZERO_DATE：  
在严格模式，不要将 '0000-00-00'做为合法日期。你仍然可以用IGNORE选项插入零日期。在非严格模式，可以接受该日期，但会生成警告  
ERROR_FOR_DIVISION_BY_ZERO：  
在严格模式，在INSERT或UPDATE过程中，如果被零除(或MOD(X，0))，则产生错误(否则为警告)。如果未给出该模式，被零除时MySQL返回NULL。如果用到INSERT IGNORE或UPDATE IGNORE中，MySQL生成被零除警告，但操作结果为NULL。  
NO_AUTO_CREATE_USER  
防止GRANT自动创建新用户，除非还指定了密码。  
NO_ENGINE_SUBSTITUTION：  
如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常  
- 据说是MySQL5.0以上版本支持三种sql_mode模式：ANSI、TRADITIONAL和STRICT_TRANS_TABLES。   
1. ANSI模式：宽松模式，更改语法和行为，使其更符合标准SQL。对插入数据进行校验，如果不符合定义类型或长度，对数据类型调整或截断保存，报warning警告。对于本文开头中提到的错误，可以先把sql_mode设置为ANSI模式，这样便可以插入数据，而对于除数为0的结果的字段值，数据库将会用NULL值代替。  
将当前数据库模式设置为ANSI模式：  
 set @@sql_mode=ANSI;    
2. TRADITIONAL模式：严格模式，当向mysql数据库插入数据时，进行数据的严格校验，保证错误数据不能插入，报error错误，而不仅仅是警告。用于事物时，会进行事物的回滚。 注释：一旦发现错误立即放弃INSERT/UPDATE。如果你使用非事务存储引擎，这种方式不是你想要的，因为出现错误前进行的数据更改不会“滚动”，结果是更新“只进行了一部分”。  
将当前数据库模式设置为TRADITIONAL模式：  
 set @@sql_mode=TRADITIONAL;     
 3. STRICT_TRANS_TABLES模式：严格模式，进行数据的严格校验，错误数据不能插入，报error错误。如果不能将给定的值插入到事务表中，则放弃该语句。对于非事务表，如果值出现在单行语句或多行语句的第1行，则放弃该语句。  
将当前数据库模式设置为STRICT_TRANS_TABLES模式：  
 set @@sql_mode=STRICT_TRANS_TABLES;    
另外说一点，这里的更改数据库模式都是session级别的，一次性，关了再开就不算数了！！！  
也可以通过配置文件设置:vim /etc/my.cnf  
在my.cnf（my.ini）添加如下配置:  
[mysqld]  
sql_mode='你想要的模式'  

# 二进制命令  
二进制日志（binary log）记录了对MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。  
SHOW BINLOG EVENTS IN 'mysql-bin.000001'  
binlog_format  
row  statement mixed三种格式RBR SBR MBR  
在row格式下，开启binlog_rows_query_log_events参数  
- 二进制日志的删除  
服务器自动清理旧的binlog文件，需设置expire-logs-days选项  
purge binary logs before datetime  
purge binary logs to 'filename'  
- 二进制日志过滤器  
[mysqld]  
binlog-ignore-db=one_db  
binlog-do-db=two_db  
Mysql实现企业级日志管理、备份与恢复的实战教程 http://www.jb51.net/article/130069.htm  
  
  
# 复制、备份与恢复  
mysqldump -uroot -p --databases databasename >dump.sql 数据库备份  
mysqldump -uroot -p --databases database.table >dump.sql 数据库表备份  
load data infile '/home/mysql/film_test.txt' into table tablename;  
alter table film_test2 disable keys;  
load data infile '/home/mysql/film_test.txt' into table tablename;  
alter table film_test2 enable keys;  
  
# 主从复制的服务器配置  
1. 主服务器  
log-bin=mysql-bin  
sever-id=**  
2. 从服务器  
log-bin=mysql-bin  
sever-id=**  
3. 重启两台服务器的mysql  
4. 主服务器建立授权  
grant replication slave on *.* to 'username'@'host' identified by 'password';  
5. show master status;记录master_log_file,master_log_pos  
6. stop slave   
change master to master_host='192.168.203.129',master_user='mysync',master_password='q123456',master_log_file='mysql-bin.000017',master_log_pos=107;  
start slave  
7. show slave status  
  
[mysql5.7多源复制](http://www.cnblogs.com/xuanzhi201111/p/5151666.html)  
[mysql-master/slave同步问题：Slave_IO_Running: No](http://www.51testing.com/html/00/130600-243651.html)  
  
  
# 开发规范  
表设计的规范：字段数量建议不超过20-50个做好数据评估，建议纯INT不超过1500万，含有CHAR的不要超过1000万。字段类型在满足需求条件下越小越好，尽量使用UNSIGNED存储非负整数，因为实际使用时候存储负数的场景不多。将字符转换成数字存储。例如使用UNSIGNED INT存储IPv4 地址而不是用CHAR(15) ，但这种方式只能存储IPv4，存储不了IPv6。另外可以考虑将日期转化为数字，如：from_unixtime()、unix_timestamp()。所有字段均定义为NOT NULL，除非你真的想存储null。  
  
Schema设计原则核心表字段数量尽可能地少，有大字段要考虑拆分适当考虑一些反范式的表设计，增加冗余字段，减少JOIN 资金字段考虑统一*100处理成整型，避免使用decimal浮点类型存储日志类型的表可以考虑按创建时间水平切割，定期归档历史数据  
  
垂直拆分  
优点：拆分简单明了，拆分规则明确应用程序模块清晰，整合容易数据维护方便易行，容易定位  
缺点：表关联需要改到程序中完成事务处理变的复杂热点表还有可能存在性能瓶颈过度拆分会造成管理复杂  
水平拆分  
优点：不会影响表关联、事务操作超大规模的表和高负载的表可以打散应用程序端改动比较小拆分能提升性能，也比较易扩展  
缺点：数据分散，影响聚集函数的使用切分规则复杂，维护难度增加后期迁移较复杂  
  
# 需要监控的参数  
OS: CPU ,Mem ,Net, IO ,Swap  
MySQL: Network,Usage,Com*,Connection, Open Table/Files,Temporary Questions  
InnoDB: written, read,dirty writes/reads,flush,innodb_rows,innodb_log,innodb_buffer_pool_pages  
doing:processlist,engine innodb status,threads,locks,slow queries  
监控  
linux和unix系统监控工具  
ps 显示系统上运行的进程列表 ps -A | grep mysqld         
top 显示根据cpu使用率排序的活动进程  top -d 3 每3秒刷新命令  
vmstat 显示内存、分页、块传输和CPU活动的相关信息  
uptime 显示系统运行了多长时间  
free 显示内存使用率  
iostat 显示平均磁盘活动和处理器负载情况  
sar 显示系统活动报告  
pmap 显示各种进程分别占用内存的情况  
mpstat 显示多处理器系统的CPU使用率  
netstat 显示网络活动的相关信息  
cron 可以让你安排进程执行的子系统  
  
sys  
SELECT table_schema,table_name,io_read_requests+io_write_requests AS tot FROM schema_table_statistics;  #查看表的访问量  
SELECT * FROM schema_redundant_indexes; #冗余索引与未使用索引检查  
SELECT * FROM schema_unused_indexes;  
SELECT * FROM schema_auto_increment_columns; #表自增id监控  
SELECT* FROM statements_with_full_table_scans WHERE db='zhongguo' #监控全表扫描的sql语句  
SELECT FILE,avg_read+avg_write AS avg_io FROM io_global_by_file_by_bytes ORDER BY avg_io DESC LIMIT 10 #查看实例消耗的磁盘io  
  
sysbench   
https://blog.csdn.net/mysql_lover/article/details/54139837  
http://www.cnblogs.com/Aiapple/p/5702977.html  
  
# 数据库预热   
SELECT table_schema, table_name FROM information_schema.tables mysql5.1  
通常在mysql重启服务后，需要通过手工执行SQL来预热buffer_pool，在mysql5.6中，有如下参数可以无需人工干预。  
innodb_buffer_pool_dump_at_shutdown= 1:在关闭时把热数据dump到本地磁盘  
innodb_buffer_pool_dump_now = 1：采用手工方式把热数据dump到本地磁盘  
innodb_buffer_pool_load_at_startup=1：启动时把热数据加载到内存  
innodb_buffer_pool_load_now=1：采用手工方式把热数据加载到内存  
只有在正常关闭或pkill mysql是才会将热数据导出。  
  

[性能调优](http://www.cnblogs.com/chunguang/p/5694582.html)   
[详细分析SQL语句逻辑执行过程和相关语法](https://www.cnblogs.com/f-ck-need-u/p/8656828.html)


安装与升级  
[源码安装](http://blog.51cto.com/xpleaf/1748063)  
[mysql的zip安装](https://blog.csdn.net/fhq10/article/details/70157170)    
[Keepalived + 双mycat + mysql主主复制](http://www.cndba.cn/Marvinn/article/2615)    

# 存储函数和函数  
IF(Condition,A,B)  
意义：当Condition为TRUE时，返回A；当Condition为FALSE时，返回B。  
  
substring(group_contact(字段1 order by 字段2 desc),',',1) 取出按字段2倒xu排列的字段1  
substring_index('varstr',string,1) 按varstr的位置截取字符串string，1顺序，-1倒序  
postion(),locate(),instr() 返回字符在字段中的位置  

# 技巧  
1. 行转列  
CREATE TABLE `score` (  
  `name` varchar(20) default NULL,  
  `course` varchar(20) default NULL,  
  `grade` int(11) default NULL  
) ENGINE=InnoDB DEFAULT CHARSET=utf8  
select DISTINCT c.`name` as  name,(select grade from score where name=c.`name` and course='java') as java,  
(select grade from score where name=c.`name` and course='c') as c,  
(select grade from score where name=c.`name` and course='go')as go  
from score c  
select name ,SUM(case course when 'java' then grade end) java,  
SUM(case course when 'c' then grade end) java,  
SUM(case course when 'go' then grade end) java  
from score GROUP BY name  
  
2. 随机取数据 join效率高  
SELECT *  
FROM `table` AS t1 JOIN (SELECT ROUND(RAND() * (SELECT MAX(id) FROM `table`)) AS id) AS t2  
WHERE t1.id >= t2.id  
ORDER BY t1.id ASC LIMIT 5;  
SELECT * FROM `table`  
WHERE id >= (SELECT floor( RAND() * ((SELECT MAX(id) FROM `table`)-(SELECT MIN(id) FROM `table`)) + (SELECT MIN(id) FROM `table`)))   
ORDER BY id LIMIT 1;  
  
3. 处理重复数据  
group by  
distinct  
ALTER IGNORE TABLE person_tbl ADD PRIMARY KEY (last_name, first_name);  
DELETE t1 FROM table1 AS t1 JOIN table1 AS t2 ON t1.id>t2.id AND t1.name=t2.name;  
  
4. 交集、并集和差集用sql实现   
[链接1](http://www.cnblogs.com/jackson0714/p/TSQLFundamentals_05.html)  
[链接2](http://blog.csdn.net/GOOWJ/article/details/8728189)  
SELECT a.* FROM(SELECT id,code,name FROM test_emp UNION ALL SELECT id,code,name FROM test_emp WHERE dept='JSB')a GROUP BY a.id HAVING COUNT(a.id)=1 #差集  
SELECT a.* FROM(SELECT id,code,name FROM test_emp WHERE age>25 UNION ALL SELECT id,code,name FROM test_emp WHERE dept='JSB')a GROUP BY a.id HAVING COUNT(a.id)=2 #交集  
SELECT a.* FROM(SELECT id,code,name FROM test_emp WHERE age>25 UNION SELECT id,code,name FROM test_emp WHERE dept='JSB')a #并集  
  
5. 解决MySQL数据量增大之后翻页慢问题  
原始sql
SELECT a.* FROM event_data a WHERE a.receive_time >= '2018-03-28 00:00:00' AND a.receive_time <= '2018-03-28 23:59:59' ORDER BY a.receive_time DESC LIMIT 56280,15;    
优化sql 
select a.* FROM (SELECT pk_id FROM event_data c WHERE c.receive_time >= '2018-03-28 00:00:00' AND c.receive_time <= '2018-03-28 23:59:59' ORDER BY c.receive_time DESC LIMIT 56280,15) b left join event_data a on a.pk_id=b.pk_id  
[mysql分页查询优化](https://www.cnblogs.com/geningchao/p/6649907.html)  
SELECT * FROM 表名称 LIMIT M,N  
SELECT * FROM your_table WHERE pk>=1000 ORDER BY pk ASC LIMIT 0,20  
select a.* from tablename a join (select id from tablename limit 100000,20) b on a.id=b.id;  
SELECT * FROM your_table WHERE id <=   
(SELECT id FROM your_table ORDER BY id desc LIMIT ($page-1)*$pagesize ORDER BY id desc LIMIT $pagesize  

6. MySQL逗号分割字段的行列转换技巧  
select a.ID,substring_index(substring_index(a.mSize,',',b.help_topic_id+1),',',-1)   
from   tbl_name a  join  mysql.help_topic b  
on b.help_topic_id < (length(a.mSize) - length(replace(a.mSize,',',''))+1)  
order by a.ID;  
7. mysql实现排名函数    
* 建表  
`CREATE TABLE city_popularity(
    region int(10) NOT NULL COMMENT '1 国内 2 海外',
    city_name VARCHAR(64) NOT NULL,
    popularity DOUBLE(5,2) NOT NULL);`
* 向表中插入数据  
```
INSERT INTO city_popularity (region, city_name, popularity)
VALUES
(1, '北京', 30.0),
(1, '上海', 30.0),
(1, '南京', 10.0),
(2, '伦敦', 20.0),
(1, '张家界', 8.0),
(2, '纽约', 35.0),
(1, '三亚', 25.0),
(2, '新加坡', 35.0);
```

region | city_name | popularity  
---|---|---  
1 | 北京 | 30.0  
1 | 上海 | 30.0  
1 | 南京 | 10.0  
2 | 伦敦 | 20.0  
1 | 张家界 | 8.0  
2 | 纽约 | 35.0  
1 | 三亚 | 25.0  
2 | 新加坡 | 35.0  

对数据进行排序  
  * 通过窗口函数  
  MySQL从8.0开始支持窗口函数，也叫分析函数，序号函数ROW_NUMBER(), RANK(), DENSE_RANK()满足不同需求的排序  
  `SELECT region, city_name, popularity, 
  ROW_NUMBER() OVER (PARTITION BY region ORDER BY popularity DESC) AS rank 
  FROM city_popularity;`
  * 通过表的自交  
  `SELECT a.region, a.city_name, a.popularity, (COUNT(b.popularity)+1) AS rank 
  FROM city_popularity AS a LEFT JOIN city_popularity AS b 
  ON a.region = b.region AND a.popularity<b.popularity
  GROUP BY a.region, a.city_name, a.popularity
  ORDER BY a.region, rank;`
  * 通过设置变量  
  顺序排序，每多一条排序自增加一  
  `SELECT city_popularity.*,
  @rank := @rank+1 AS rank
  FROM city_popularity ,(SELECT @rank:=0) init
  ORDER BY popularity DESC;`  
  当数据相同时，排名一致，不相同则排名自增加一  
  `select city_popularity.*,
  case when @popularity = popularity then @rank 
  when @popularity := popularity then @rank :=@rank+1 
  when @popularity =0 then @rank :=@rank+1 END as rank 
  from city_popularity,(select @rank :=0,@popularity :=NULL) init  
  ORDER BY popularity DESC;`  
  数据相同的情况，排名保持不变，且占有字符  
  `select city_popularity.*,
  @rank1 :=@rank1+1,@rank := 
  case when @popularity = popularity then @rank 
  when @popularity := popularity then @rank1 
  when @popularity =0 then @rank1 END as rank
  from city_popularity,(select @rank :=0,@popularity :=NULL,@rank1 :=0) init  
  ORDER BY popularity DESC;`  

8. 模拟generate_series() in PostgreSQL   
`select date from (select date_format(adddate(MAKEDATE(year(now()),1), @num:=@num+1),'%Y-%m-%d') date from your_table,(select @num:=-1) num
limit 366 ) as dt `  

# 错误解决
- 当连接错误次数过多时，mysql会禁止客户机连接，这个时候有两个办法解决：  
    1. 使用mysqladmin flush-hosts命令清除缓存，命令执行方法如下：  
          命令行或终端：mysqladmin  -u  root  -p  flush-hosts  
          接着输入root账号密码  
    2. 修改mysql配置文件，在[mysqld]下面添加 max_connect_errors=1000，然后重启mysql  

    查看死锁日志  
    show engine innodb status  
- 字符串  
    char在存储的时候会将右侧空格进行剔除，保留左侧空格。  
    varchar在存储的时候保留所有空格，不进行任何删除  
    varchar和char在查询的时候都只会根据where条件中的左侧空格进行判断，右侧末尾的空格会忽略  


# 性能排查    
   服务性能排查一般就两种：高内存占用或高CPU占用，需要具体问题具体分析。比如应用程序高内存占用，可能因为大文件读取、频繁IO，内存消耗频繁，导致频繁GC，进一步占用内存和CPU；比如应用程序高CPU占用，可能在执行大任务计算，或者死循环、卡死，或者不断超时、重试（活锁是容易占CPU的，死锁和饥饿是容易占内存的，因为资源不释放）。  
应用进程还活着，但页面出不来、不响应，这种是高CPU，高内存是应用响应慢或者内存溢出、直接死掉。  
    从这两个方向考虑，比如：  
高CPU占用的话，关注一下，是user态占用率高还是sys态占用率高，闲置多少；占用率高的进程是否频繁变动；load平均负载是否过载(超过70%)。如果user态CPU占用率过高，意味着应用在做高耗CPU的任务；如果sys态占用率过高，则意味着内核态的操作比较多比如IO复制；如果占用率高的线程频繁变动，则可能是CPU时间片不断调度，线程唤醒跑一下而后换另一个线程跑，需要看多线程任务是否存在大计算问题，以及线程池设置是否合适等；  
高内存的话，关注一下，free可用内存有多少，buff、cache还有多少；swap交换区是否过高，这通常是因为未禁用swap区而内存又不足，导致swap频繁读写，性能低下；sar内存变化曲线，是稳定上升还是高低曲线，前者意味着某个函数存在内存泄漏问题，而后者意味着有某个高内存运算任务在线程池里唤醒重试；  
    更实际的情形是，应用出现高应用或者高CPU的问题时，你只有30秒到两三分钟来快速定位问题，而后就要马上重启应用，避免影响到正常业务。像高CPU，可能连日志都没有，因为请求没进来，很多时候你都需要去前后dump线程堆栈或内存堆栈，以保留现场方便后续分析工作。成熟的线上环境，一般会有仪表大盘和监控预警，内存或CPU超过阈值你就需要立马去关注了，或应用发布回滚，在线下验证问题。  
    一般出问题时，首先去看监控大盘，有没有异常告警，看不出问题来再去查看系统层面有没有异常：  
通过top或htop命令观察系统的整体情况，很容易拿到占用CPU或内存较高的进程ID；  
先通过ps aux | grep [PID]进一步确定是Java进程还是服务器上其它进程；  
如果是CPU占用高，通过pidstat查看该进程，然后用perf/trace+PID，抓取CPU消耗栈来找到引发瓶颈的具体的函数名，结合业务代码来分析问题具体在哪儿；  
通过ps -mp pid -o THREAD,tid,time 找到具体的线程ID；  
printf "%x\n" [TID]] 将线程id转换为16进制  
如果是内存占用高，通过free命令查看内存的使用情况，然后通过pmap/jmap命令查看进程的内存分布  
或是通过vmstat命令查看内存使用的变化趋势，用memleak -a -p [PID]查看内存分配栈，定位到是哪个函数内存泄漏了  
如果CPU、内存没问题，就要去看磁盘，通过df命令和iostat命令去查看磁盘空间和I/O情况；  
load平均负载和CPU使用率是有区别的，一个是单位时间内的活跃进程数，一个是单位时间内CPU空闲时间与总CPU时间的比值。它们之间有一定的联系，比如在CPU密集型应用中，大计算量任务会导致大量的CPU被占用，平均负载升高，而在I/O密集型应用中，不可中断进程的增多会导致平均负载升高，但I/O等待不占CPU，CPU使用率并不一定会很高。  
CPU性能排查时首先去查看CPU使用率，比如user用户态较高，则往应用进程的性能问题去排查；如果sys内核态较高，则往系统调用的性能问题去排查。但很多时候，监控告警之后，等你登陆服务器时性能问题已经结束了，这样在线分析就看不出问题了，就需要从load平均负载、sar历史记录去回溯  
