# 当连接错误次数过多时，mysql会禁止客户机连接，这个时候有两个办法解决：  
1. 使用mysqladmin flush-hosts命令清除缓存，命令执行方法如下：  
      命令行或终端：mysqladmin  -u  root  -p  flush-hosts  
      接着输入root账号密码  
2. 修改mysql配置文件，在[mysqld]下面添加 max_connect_errors=1000，然后重启mysql  

# 查看死锁日志  
show engine innodb status  
