<!-- TOC -->

- [命令行操作](#命令行操作)
- [权限与安全](#权限与安全)
- [临时开启通用查询日志](#临时开启通用查询日志)
- [二进制命令](#二进制命令)
- [sql_mode](#sql_mode)
- [二进制日志](#二进制日志)
- [复制、备份与恢复](#复制备份与恢复)
- [主从复制的服务器配置](#主从复制的服务器配置)
- [开发规范](#开发规范)
- [需要监控的参数](#需要监控的参数)
- [数据库预热](#数据库预热)
- [存储函数和函数](#存储函数和函数)
- [技巧](#技巧)

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
  
# 二进制命令  
二进制日志（binary log）记录了对MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。  
SHOW BINLOG EVENTS IN 'mysql-bin.000001'  
  
binlog_format  
row  statement mixed三种格式RBR SBR MBR  
在row格式下，开启binlog_rows_query_log_events参数  
  
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

# 二进制日志
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
