# 捕捉正在执行的查询  
*  方式一
```
SELECT
	pid,--pid
  usename,--执行的用户
	backend_start,--会话开始时间
	query_start,--查询开始时间
	waiting,--是否等待执行
	now() - query_start AS current_query_time,-- 累计执行时间
	now() - backend_start AS current_session_time,
	query,
	STATE,
	state_change,
	client_addr,
	datname 
FROM
	pg_stat_activity 
ORDER BY
	STATE,
	current_query_time DESC
```
*   方式二
```
SELECT     procpid,     start,     now() - start AS lap,     current_query 
FROM     (SELECT         backendid,         pg_stat_get_backend_pid(S.backendid) AS procpid,         pg_stat_get_backend_activity_start(S.backendid) AS start,       pg_stat_get_backend_activity(S.backendid) AS current_query 
    FROM         (SELECT pg_stat_get_backend_idset() AS backendid) AS S     ) AS S WHERE    current_query <> '<IDLE>' ORDER BY    lap DESC;
```
# 终止执行的sql  
`select pg_terminate_backend(48988); --pid`

# 显示表的大小  
`SELECT sotdoid,sotdsize/1024/1024/1024 as sotdsize,sotdtoastsize,sotdadditionalsize,sotdschemaname,sotdtablename from gp_toolkit.gp_size_of_table_disk order by sotdsize desc;`

# 查看行数 
```
SELECT relname, reltuples 
FROM pg_class r JOIN pg_namespace n 
ON (relnamespace = n.oid) 
WHERE relkind = 'r' AND n.nspname = 'calc';
```

# 查看建表时间
`SELECT s.oid,s.relname,t.stausename,t.stasubtype,t.statime FROM pg_class s,pg_stat_last_operation t WHERE  t.objid = s.oid ORDER  BY t.statime`

# 删除重复行  
Greenplum/PostgreSQL中数据表数据去重的几种方法  
对于在PostgreSQL中，唯一确定一行的位置的是用ctid,可以用这个ctid作为一行的唯一标识；在Oracle中，数据表中的一行的唯一标识可以使用ROWID进行标识，这作为这一行的物理地址信息。而在GP中，要唯一的标识出一行表数据，需要使用gp_segment_id加上ctid进行标识。 gp_segment_id代表的是GP的segment的节点标识，每个子库的标识是唯一的  
这种语句适合所有的GP表，特别对那种没有唯一主键的数据仓库的表进行去重很有用。  
* greenplum去重  
	```
	delete from tablename where gp_segment_id::varchar(100)||ctid::varchar(100) in
	(select t.ctid from
	(select gp_segment_id::varchar(100)||ctid::varchar(100) as ctid,a.cloumn1,a.column2,a.column3,
	row_number() over (partition by a.cloumn1,a.column2,a.column3) rows_num
	from tablename a  ) t
	where t.rows_num >=2);
	```  
	```
	delete from tablename where (gp_segment_id,ctid) in
	(select t.gp_segment_id,t.ctid from
	(select gp_segment_id,ctid,a.cloumn1,a.column2,a.column3,
	row_number() over (partition by a.cloumn1,a.column2,a.column3) rows_num
	from tablename a ) t
	where t.rows_num >=2);  
	```  

* postgresql中去重   
```
	create table vertent (id bigint);  
	
	insert into vertent values (1),(2),(2),(3);    
	
	delete from vertent where ctid in
	(select ctid from
	(select ctid,id,
	row_number() over (partition by id) rows_num
	from vertent ) t
	where t.rows_num >=2);
```
	
* oracle去重    
```
	delete from tablename  where ROWID in
	(select ROWID from
	(select ROWID,a.cloumn1,a.column2,a.column3,
	row_number() over (partition by a.cloumn1,a.column2,a.column3) rows_num
	from tablename a  ) t
	where t.rows_num >=2);
```
# copy导入导出数据  
```
导入  
psql -Ugpadmin -W -h192.168.1.28 -p12345 -dgpdpot -c "copy edge_author_2016(src,dest) from stdin with delimiter ','  NULL 'null_string' "  <  /opt/oracledata/old_re/dege_author_2016.csv  
导出  
psql -Ugpadmin -W -h192.168.1.28 -p12345 -dgpdpot  -c "copy tablename from stdout " > /opt/dpot/data/dt/pagerank.csv  
直接读取密码
PGPASSWORD=xxxx psql -h192.168.1.1 -Uxxxx  -p5432 -d xx -c "copy xx.xx to stdout" > path
```
# 外部表  
```
CREATE SERVER mysql_server FOREIGN DATA WRAPPER mysql_fdw OPTIONS (host '192.168.1.25', port '3306');

CREATE USER MAPPING FOR gpadmin --greenplum登录账户
SERVER mysql_server  
OPTIONS (username 'uesrname', password '123456');  

CREATE FOREIGN TABLE userinfo (id int8,name text) server mysql_server options (dbname 'dpot_web', table_name 'user');  

select * from userinfo
```
