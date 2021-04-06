## 第一章 安装启动MySQL

##### yum安装

```bash
#安装mysql-5.6
rpm -vih http://repo.mysql.com/mysql57-community-release-el7.rpm
yum makecache fast
yum install -y mysql-community-server
/etc/init.d/mysqld start

#安装mysql-5.7
rpm -vih http://repo.mysql.com/mysql57-community-release-el7.rpm
yum makecache fast
yum install -y mysql-community-server
systemctl start mysqld
grep log /etc/my.cnf
grep 'password' /var/log/mysqld.log				=>sqox7So3<jYe
mysql -p
set password=PASSWORD('*********');

#mysql8改密码
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'test';				//mysql8.0改密码
create user 'root'@'%' identified by 'test';
flush privileges;
grant all privileges on *.* to 'test1'@'%' with grant option;

```

##### 二进制安装

```bash
二进制安装默认在/usr/local/目录下，如果主程序安装在了其它目录(如：/app/mysql)就需要修改这两个启动脚本才能正常启动
sed -i.bak 's#/usr/local#/app#g' /app/mysql/bin/mysqld_safe
sed -i.bak 's#/usr/local#/app#g' /app/mysql/support-files/mysql.server 

1.先下载安装
yum remove -y mariadb* 
wget https://downloads.mysql.com/archives/get/file/mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz
tar xf mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz  -C /usr/local/
mv mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz mysql-5.7.20
ln -s mysql-5.7.20 mysql

2.初始化数据库
useradd mysql -s /sbin/nologin -M
cd /usr/local/mysql/ && ./bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --explicit_defaults_for_timestamp
注：--initialize-insecure 初始化并关闭安全防护,-insecure意思是初始化时不生成root用户的随机密码

3.启动数据库
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
/etc/init.d/mysqld start
echo 'export PATH="/usr/local/mysql/bin:$PATH"'>/etc/profile.d/mysql.sh
source /etc/profile
mysql

如果没有cp到init.d下就需要这样启动：cd /usr/local/mysql/support-files/ && ./mysql.server start
```

##### 源码安装

```mysql
1.下载安装
wget https://downloads.mysql.com/archives/get/file/mysql-5.6.44.tar.gz
tar xf mysql-5.6.44.tar.gz -C /application
ln -s /application/mysql-5.6.44 /application/mysql
mkdir /application/mysql/socket
yum install -y gcc gcc-c++ glic cmake ncurses-devel autoconf
cmake . -DCMAKE_INSTALL_PREFIX=/application/mysql \
-DMYSQL_DATADIR=/application/mysql/data \
-DMYSQL_UNIX_ADDR=/application/mysql/socket/mysql.sock \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_EXTRA_CHARSETS=all \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITH_ZLIB=bundled \
-DWITH_SSL=bundled \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DENABLE_DOWNLOADS=1 \
-DWITH_DEBUG=0
make && make install

2.配置并启动
useradd mysql -s /sbin/nologin -M
chown -R mysql.mysql /application/mysql
cd /application/mysql && ./mysql_install_db --user=mysql --basedir=/application/mysql
cp /application/mysql/support-files/mysql.server /etc/init.d/mysqld
cp /application/mysql/support-files/my-default.cnf /etc/my.cnf
/etc/init.d/mysqld  start
echo 'export PATH="/application/mysql/bin:$PATH"' >/etc/profile.d/mysql.sh
source /etc/profile
mysql

3.使用systemd管理mysql
[root@db01 ~]# vim /usr/lib/systemd/system/mysqld.service
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=https://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/application/mysql/bin/mysqld --defaults-file=/etc/my.cnf
LimitNOFILE = 5000
systemctl daemon-reload
systemctl start mysqld
systemctl enable mysqld
```

## 第二章 Mysql启停连接

##### 起停mysql

```bash
mysql自带的启动脚本有两个：
/usr/local/mysql/support-files/mysql.server		=>会cd到主程序目录下调用bin/mysqld_safe
/usr/local/mysql/bin/mysqld_safe							=>安全模式，可指定参数多用于维护，也可管理多实例

#mysql.server用法
一般会把mysql.server拷贝到/etc/init.d下，省得每次要进到support-files目录里来启动
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld.sh
/etc/init.d/mysqld {start|stop|restart|reload|force-reload|status}

#mysqld_safe用法
mysqld_safe --default-file=/etc/my.cnf_test &								#启动时指定配置文件,此时不会读取所有my.cnf文件
mysqld_safe --skip-grant-tables --skip-networking						#忘记root密码时跳过认证
mysqld_safe --datadir=/application/mysql/data
mysqld_safe --basedir=/application/mysql
mysqld_safe --pid-file=/application/mysql/data/db01.pid
mysqld_safe --socket=/application/mysql/data/mysql.sock
mysqld_safe --user=mysql
mysqld_safe --port=3306
mysqld_safe --log-error=/application/mysql/data/db01.err

注意：用systemctl start mysql启动mysql后无法通过/etc/init.d/mysqld stop关闭
```

##### 连接mysql

```bash
连接mysql有两种方式：
socket连接	=> 套接字连接，仅支持本地连接，所以安全且速度快，是mysql默认的连接方式；
tcp/ip连接  => 远程连接，如果连接命令中同时指定远程连接和socket连接，mysql优先使用tcp连接
mysql> show processlist;							#通过HOST查看当前连接方式

mysql -uroot -S /tmp/mysql.sock -p		#-S指定socker,如果未在配置文件中指定socket则默认在/tmp/mysql.sock
mysql -uroot -h127.0.0.1 -p -P				#-h指定远程的mysql主机 -P(大写)是指定端口 -p是要求密码 
```

##### 配置文件优先级

```bash
最高是命令行直接指定配置	=>	mysqld_safe --socket=/tmp/mysql.sock
其次是命令指定的配置文件	=>	mysqld_safe --default-file=~/my.cnf_jonny
再次是系统默认的配置文件	=>  后读取的配置文件依次覆盖前一个配置文件里不同的配置
最低是编译安装预编译选项	=>	-DSYSCONFDIR=/etc 

系统默认配置文件读取顺序如下,可通过命令查看：mysqld --help --verbose |grep my.cnf
/etc/my.cnf -> /etc/mysql/my.cnf -> /usr/local/mysql/my.cnf -> ~/.my.cnf
```

##### 配置文件

```bash
[root@db01 ~]# cat /etc/my.cnf

[mysqld]
basedir = /application/mysql
datadir = /application/mysql/data
socket=/application/mysql/tmp/mysql.sock
character-set-server=utf8

#日志管理
log-bin = mysql-bin
binlog_format = row
general_log=on
general_log_file=/application/mysql/data/db01.log
log_error=/application/mysql/data/db01.err
slow_query_log=1
slow_query_log_file=/application/mysql/data/slow.log
long_query_time=5
log_queries_not_using_indexes

#事务管理
autocommit=1

#表空间管理
innodb_data_file_path=ibdata1:12M;ibdata2:100M:autoextend

[client]
socket=/application/mysql/tmp/mysql.sock
```

## 第三章 用户管理

##### 对用户的增删改查

```bash
mysql的用户是用 username@'主机域'来表示，主机域指用户可通过哪些ip远程连接到mysql，常见主机域有：
'10.0.0.51' '10.0.0.5%' '10.0.%.%'  '%' 	#10.0.0.5%可表示10.0.0.5，'%'表示内网外网里所有主机
'10.0.0.51/255.255.255.0'									#支持子网掩码，但不支持0.0.0.0/24的写法

#添加用户并授权
grant all on *.* to zhouxy@'10.0.0.%' identified by '123';	#新建用户并授权，8.0不支持授权时建用户
create user dev@'10.0.0.%' identified by '123';							#8.0版本后需要先建用户
grant select,create on tlbb.* to dev@'10.0.0.%';						#再给授权
grant all on *.* to root@'localhost' with grant option;			#赋予超级权限，可管理其它用户的权限

#删用户
drop user zhouxy@'10.0.0.%',test@'%';												#删除多个用户

#查看用户信息
select user,host,authentication_string from mysql.user;			#查看用户名主机域和密码信息
select * from mysql.user where user='root'\G								#查看root用户所有信息

#改用户密码
set password=password('123');																#设置当前用户密码
set password for zxy@'%' =password('111');									#为其它用户设密码
update mysql.user set host='10.0.20.%' where user='dev';		#修改用户名
rename user zhou@'%' to zhouxy@'%';													#修改用户名
flush privileges;																						#修改完用户要刷新授权表，改密码时不需要
注意：当修改完用户名后，需要手动重载授权表，否则系统不识别新用户名，无法通过新用户名对该用户做任何操作，此时旧用户依然在授权表里，还可以用。
```

##### 用户权限管理

```bash
#给用户授权要遵循权限最小化原则，最小权限是单列级别，另外给权限时要脱离敏感信息如手机身份证等。常见权限如下：
alter: 修改已存在的数据表(例如增加/删除列)和索引
create: 建立新的数据库或数据表
delete: 删除表的记录
drop: 删除数据表或数据库
index: 建立或删除索引
insert: 增加表的记录
select: 查看表的记录
update: 修改表中已存在的记录
file: 在MySQL服务器上读写文件
process: 显示或杀死属于其它用户的服务线程
reload: 重载访问控制表，刷新日志等
shutdown: 关闭MySQL服务 
usage: 只允许登录，别的都不行
all: 所有权限，普通管理员
with grant option: 超级管理员，可以给别的用户授权

#操作语句
grant select(user,host) on mysql.user to dev@'%';				#授权用户dev只能查看user列和host列的内容
revoke select(host) on *.* from dev@'%';								#掉去查看host列的权限
revoke all on *.* from dev@'%';													#取消用户dev的所有权限，只剩下usage,仅仅能登陆
show grants for dev@'%';																#查看用户所有权限
grant all on *.* to zxy@'%' with max_user_connection 3; #除基本权限外的其它限制

with max_queries_per_hour 180														#允许用户每小时最多可发起180个查询操作
with max_updates_per_hour 180														#允许用户每小时最多可发起180个更新操作
with max_connetions_per_hour 8													#允许用户每小时最多可连接到服务器8次
with max_user_connetions 3															#允许用户最多同时用3个终端连接服务器，降低盗号风险
```

##### 误删root或忘记root密码

```bash
当误删了root用户或忘记root密码，不要慌。mysql的连接层是依据授权表验证用户名密码白名单等，所以只需要在mysql服务器上跳过权限表登陆，登陆后新建好root用户，再刷新授权表，重启mysql就行了。

#授权表相关操作
mysqld_safe --skip-grant-tables --skip-networking &	#跳过授权表登陆，此时可无验证登陆mysql，授权表是空的
select * from information_schema.user_privileges;		#查看授权表
flush privileges																		#当需要用grant命令时，需要先载入授权表
update mysql.user ...后 flush privileges 					 #当update时，不需先载入授权表，但修改mysq.user后要刷新
																										#..而使用grant或set password后，系统自动刷新授权表

#误删root用户
1.模拟用户被删
delete from mysql.user;															#清空mysql.user表，root用户被删
mysql> select user,host from mysql.user;						#user表已经空了
Empty set (0.00 sec)
2.解决办法
/etc/init.d/mysqld.sh  stop													#换种方式启动mysql，所有要先停mysql
mysqld_safe --skip-grant-tables --skip-networking &	#进入安全模式，跳过授权表 禁用网络(不能远程登陆，安全)
mysql																								#进入mysql
mysql> flush privileges;														#手动把授权表加载到内存，不加载的话无法执行授权相关操作
mysql> grant all on *.* to root@'localhost' identified by '111' with grant option;
/etc/init.d/mysqld.sh  restart											#重启mysql
mysql -uroot -p111																	#经验证，可用新密码登陆mysql了

#忘记root密码
/etc/init.d/mysqld.sh  stop
mysqld_safe --skip-grant-tables --skip-networking &
update mysql.user set authentication_string=password('111') where user='root' and host='localhost';
flush privileges;
\q
kill -9 4118																				#杀掉安全模式的mysql
/etc/init.d/mysqld.sh  start
mysql -p	=>input password: 111											#成功改掉密码
```

## 第四章 SQL语句

##### 建库建表规范

```bash
#建库规范
create database if not exists students charset utf8 collate uft8_general_ci;

1.建库要指定字符集，因为二进制安装默认是拉丁字符集不识别中文，字符集只作用于表的内容
2.库名全用小写字母：库名表名存放在磁盘上，库名与表名是区分大小写的，linux系统区分大小写的，unix不区分，windows不区分
3.库名要与业务相关且不可用数字开头

#建表规范
1.建表要指定字符集和存储引擎
2.表名要与业务相关，要小写且不可以数字开头
3.选择合适的数据类型避免浪费空间
4.设主键自增长，每个列都设为非空，无法保证时用0代替，必须注释

示例：
mysql> create table stu(
    -> id int(5) zerofill  primary key auto_increment comment '学号',
    -> sname varchar(12) not null comment '姓名',
    -> sage tinyint unsigned comment '年龄',
    -> sgender enum('m','f','n') default 'n' comment '性别',
    -> sid char(18) not null unique comment '身份证',
    -> intime  datetime not null default now() comment '报道时间'
    -> )engine=innodb charset=utf8 comment '学生表';
```

##### 查询语句

```bash





```



















