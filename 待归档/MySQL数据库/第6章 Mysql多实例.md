## 什么是多实例

多实例就是在同一台机器里起多个myslq服务，分别使用不同端口，主要是用于测试环境

```shell
1.多套进程+线程+内存结构 
2.多个配置文件，分别设置
	a)多个端口
	b）多个socket
	c）多个日志文件
	d）多个service_id
3.多套数据
```

## 多实例实战

##### 1.创建目录和配置文件

```shell
#创建多实例的目录如下
[root@db01 ~]# tree /app
/app
├── 3307
│   └── my.cnf
├── 3308
│   └── my.cnf
└── 3309
    └── my.cnf

#修改配置文件如下
[root@db01 /app]# cat 3307/my.cnf 
[mysqld]
basedir=/application/mysql
datadir=/app/3307/data
socket=/app/3307/data/mysql.sock
port=3307
log_error=/app/3307/data/3307.err
server_id=7
pid_file=/app/3307/data/3307.pid
-----------------------------------------------
[root@db01 /app]# cat 3308/my.cnf 
[mysqld]
basedir=/application/mysql
datadir=/app/3308/data
socket=/app/3308/data/mysql.sock
port=3308
log_error=/app/3308/data/3308.err
server_id=8
pid_file=/app/3308/data/3308.pid
-----------------------------------------------
[root@db01 /app]# cat 3309/my.cnf 
[mysqld]
basedir=/application/mysql
datadir=/app/3309/data
socket=/app/3309/data/mysql.sock
port=3309
log_error=/app/3309/data/3309.err
server_id=9
pid_file=/app/3309/data/3309.pid

```

##### 2.初始化多实例目录

```shell
#初始化所有实例
[root@db01 /application/mysql/scripts]# ./mysql_install_db  --defaults-file=/app/3307/my.cnf --user=mysql --basedir=/application/mysql  --datadir=/app/3307/data

[root@db01 /application/mysql/scripts]# ./mysql_install_db  --defaults-file=/app/3308/my.cnf --user=mysql --basedir=/application/mysql  --datadir=/app/3308/data

[root@db01 /application/mysql/scripts]# ./mysql_install_db  --defaults-file=/app/3309/my.cnf --user=mysql --basedir=/application/mysql  --datadir=/app/3309/data

```

##### 3.启动多实例

```shell
[root@db01 ~]# mysqld_safe --defaults-file=/app/3307/my.cnf &
[root@db01 ~]# mysqld_safe --defaults-file=/app/3308/my.cnf &
[root@db01 ~]# mysqld_safe --defaults-file=/app/3309/my.cnf &

[root@db01 ~]# netstat -lnutp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      6725/sshd           
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      6864/master         
tcp6       0      0 :::3306                 :::*                    LISTEN      6723/mysqld         
tcp6       0      0 :::3307                 :::*                    LISTEN      8037/mysqld         
tcp6       0      0 :::3308                 :::*                    LISTEN      8207/mysqld         
tcp6       0      0 :::3309                 :::*                    LISTEN      8377/mysqld         
tcp6       0      0 :::22                   :::*                    LISTEN      6725/sshd           
tcp6       0      0 ::1:25                  :::*                    LISTEN      6864/master  
```

##### 4.连接mysql

```shell
#设置密码
[root@db01 ~]# mysqladmin -uroot -p -S /app/3307/data/mysql.sock password '3307'
[root@db01 ~]# mysqladmin -uroot -p -S /app/3308/data/mysql.sock password '3308'
[root@db01 ~]# mysqladmin -uroot -p -S /app/3309/data/mysql.sock password '3309'

#连接
[root@db01 ~]# mysql -uroot -p3307 -S /app/3307/data/mysql.sock 
[root@db01 ~]# mysql -uroot -p3308 -S /app/3308/data/mysql.sock 
[root@db01 ~]# mysql -uroot -p3309 -S /app/3309/data/mysql.sock 

```

##### 5.简化连接

```shell
#简化连接技巧
[root@db01 /usr/bin]# vim mysql-3307 
mysql -uroot -p3307 -S /app/3307/data/mysql.sock
[root@db01 /usr/bin]# chmod +x mysql-3307 

#再连接
[root@db01 ~]# mysql-3307 
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.6.44 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```





