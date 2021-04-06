## 安装方式及区别

```shell
#源码安装
自定义各文件存放目录，更规范
一般公司需要做二次开发，增删功能需要用到源码安装
当需要批量安装时，用源码安装，再把依赖文件一起打成rpm包
#二进制安装
生产中常用，安装简单，解压即可安装上
默认安装在/usr/local目录下，可以通过修改../mysqld_safe 和/etc/init.d/mysqld文件来修改
```

## 二进制安装Mysql-5.7.20

安装前需要卸载系统自带的mariadb-5.5：yum remove -y mariadb* 

##### 第一步：下载并解压mysql

```shell
#下载
[root@db01 ~]# wget https://downloads.mysql.com/archives/get/file/mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz

#解压到创建好的目录
[root@db01 ~]# tar xf mysql-5.7.20-linux-glibc2.12-x86_64.tar.gz
[root@db01 ~]# mkdir /app
[root@db01 ~]# mv mysql-5.7.20-linux-glibc2.12-x86_64 /app/mysql-5.7.20
[root@db01 ~]# cd /app
[root@db01 /app]# ln -s mysql-5.7.20/ mysql
```

##### 第二步：初始化

```shell
#需要提前创建用户，data目录会自动创建，权限是mysql.mysql
[root@db01 /app/mysql]# useradd mysql -sr /sbin/nologin -m
[root@db01 /app/mysql]# ./bin/mysqld --initialize-insecure  --user=mysql --basedir=/app/mysql --datadir=/app/mysql/data

-insecure:关闭安全防护
------------------------------------------优化 -----------------------------------------------
#这里可以把/app/mysql/bin目录加入环境变量，再重载环境变量就可以再命令行直接调用，不需要用绝对路径
[root@db01 /app/mysql]# vim /etc/profile.d/mysql.sh 
export PATH="/app/mysql/bin:$PATH"
[root@db01 /app/mysql]# source /etc/profile
```

##### 第三步：启动数据库

```shell
#启动
[root@db01 /app/mysql/support-files]# ./mysql.server  start
#进入数据库
[root@db01 /app/mysql]# mysql

------------------------------------------优化1 -----------------------------------------------
二进制安装默认把mysql装在/usr/local/目录下,需要修改启动文件
[root@db01 /app/mysql]# vim bin/mysqld_safe 
[root@db01 /app/mysql/support-files]# vim mysql.server 
:%s#/usr/local#/app#g


------------------------------------------优化2 -----------------------------------------------
#这里可以把mysql.server拷贝到/etc/init.d/下，就可以用server方式管理mysql,而且可以加入开机自启
[root@db01 /app/mysql]# cp support-files/mysql.server /etc/init.d/mysqld
[root@db01 ~]# service  mysqld start 或 /etc/init.d/mysqld start


#使用systemd来管理mysql
[root@db01 /app/mysql]# vim /usr/lib/systemd/system/mysqld.service
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
ExecStart=/app/mysql/bin/mysqld --defaults-file=/etc/my.cnf
LimitNOFILE = 5000

[root@db01 ~]# systemctl daemon-reload 
------------------------------------------优化3 -----------------------------------------------
#修改mysql命令提示符
[root@db01 /app/mysql]# cat /etc/my.cnf 
[mysql]
prompt=db01 [\\d]>\_ 
```



## 源码安装mysql-5.6.44

##### 1.准备工作

```shell
#下载源码包
[root@db01 ~]# wget https://downloads.mysql.com/archives/get/file/mysql-5.6.44.tar.gz

#解压源码包
[root@db01 ~]# tar xf mysql-5.6.44.tar.gz

#安装C环境
[root@db02 mysql-5.6.44]# yum install -y gcc gcc-c++ glic

#安装依赖包
[root@db01 ~]# yum install -y cmake ncurses-devel autoconf
```

##### 2.安装

```shell
#创建安装目录
[root@db01 ~]# mkdir /application

#生成makefile文件（确定编译规则）
[root@db01 ~]# cd mysql-5.6.44
[root@db01 ~/mysql-5.6.44]# cmake . -DCMAKE_INSTALL_PREFIX=/application/mysql-5.6.44 \
-DMYSQL_DATADIR=/application/mysql-5.6.44/data \
-DMYSQL_UNIX_ADDR=/application/mysql-5.6.44/tmp/mysql.sock \
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

#在当前目录下找makefile文件，并依据此文件把汇编语言编译成二进制文件
[root@db01 ~/mysql-5.6.44]# make 

#把编译好的二进制语言安装进操作系统
[root@db01 ~/mysql-5.6.44]# make install
```

##### 3.配置并启动

```shell
#创建启动用户
[root@db01 mysql-5.6.44]# useradd mysql -s /sbin/nologin -M

#拷贝启动脚本
[root@db01 mysql-5.6.44]# cd /application/mysql-5.6.44/support-files/
[root@db01 /application/mysql-5.6.44/support-files]# cp mysql.server  /etc/init.d/mysqld

#拷贝配置文件
[root@db01 /application/mysql-5.6.44/support-files]# cp my-default.cnf  /etc/my.cnf
cp: overwrite ‘/etc/my.cnf’? y(覆盖)

#创建socket文件存放目录
[root@db01 /application/mysql-5.6.44]# mkdir tmp

#软链接
[root@db01 scripts]# ln -s /application/mysql-5.6.44 /application/mysql

#统一授权
[root@db01 /application]# chown -R mysql.mysql /application/mysql

#初始化数据
[root@db01 /application/mysql-5.6.44/scripts]# ./mysql_install_db  --user=mysql --basedir=/application/mysql --datadir=/application/mysql/data

#启动mysql
[root@db01 /application/mysql-5.6.44/scripts]# /etc/init.d/mysqld  start
```

##### 4.添加环境变量

```shell
#添加环境变量
[root@db01 ~]# vim /etc/profile.d/mysql.sh
export PATH="/application/mysql/bin:$PATH"

#加载环境变量
[root@db01 ~]# source /etc/profile

#检查环境变量
[root@db01 ~]# echo $PATH
/application/mysql/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin

#检查端口没问题
[root@db01 ~]# netstat -lntup
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name            
tcp6       0      0 :::3306                 :::*                    LISTEN      24241/mysqld  

#再启动mysql
[root@db01 ~]# mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.44 Source distribution

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

##### 5.使用systemd管理mysql

```shell
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

#重载systemd配置
[root@db01 ~]# systemctl daemon-reload

#加入开机自启
[root@db01 ~]# systemctl enable mysqld
Created symlink from /etc/systemd/system/multi-user.target.wants/mysqld.service to /usr/lib/systemd/system/mysqld.service.
```



## 二进制安装mysql-5.6.44

解压开就能使用，绿色安装

##### 1.安装mysql

```shell
#下载
[root@db02 ~]# wget https://downloads.mysql.com/archives/get/file/mysql-5.6.44-linux-glibc2.12-x86_64.tar.gz

#解压
[root@db02 ~]# tar xf mysql-5.6.44-linux-glibc2.12-x86_64.tar.gz

#创建安装目录
[root@db02 ~]# mkdir /application

#移动解压后的安装包到安装目录
[root@db02 ~]# mv mysql-5.6.44-linux-glibc2.12-x86_64 /application/mysql-5.6.44

#做软链接
[root@db02 ~]# ln -s /application/mysql-5.6.44/ /application/mysql
```

##### 2.配置mysql

```shell
#创建启动用户
[root@db02 ~]# useradd mysql -s /sbin/nologin -M

#拷贝启动脚本
[root@db02 /application/mysql/support-files]# cp mysql.server  /etc/init.d/mysqld

#拷贝配置文件
[root@db02 /application/mysql/support-files]# cp my-default.cnf /etc/my.cnf
cp：是否覆盖"/etc/my.cnf"？ y

#初始化
[root@db02 /application/mysql/scripts]# yum install -y libaio-devel
[root@db02 /application/mysql/scripts]# yum install -y autoconf
[root@db02 /application/mysql/scripts]# ./mysql_install_db  --user=mysql --basedir=/application/mysql  --datadir=/application/mysql/data

#授权
[root@db02 scripts]# chown -R mysql.mysql /application/mysql*
```

##### 3.启动mysql

```shell
#修改启动脚本和程序
[root@db02 /application/mysql/scripts]# sed -i "s#/usr/local#/application#g" /etc/init.d/mysqld /application/mysql/bin/mysqld_safe

#启动
[root@db02 /application/mysql/scripts]# /etc/init.d/mysqld start
Starting MySQL.Logging to '/application/mysql/data/db02.err'.
 SUCCESS! 
```

##### 4.添加环境变量

```bash
[root@db02 /application/mysql/scripts]# vim /etc/profile.d/mysql.sh
export PATH="/application/mysql/bin:$PATH"

[root@db02 /application/mysql/scripts]# source  /etc/profile
```

##### 5.使用systemd管理

```shell
#systemd管理mysql启动
[root@db02 mysql-5.6.36]# vim /usr/lib/systemd/system/mysqld.service
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

#修改配置文件
[root@db02 ~]# vim /etc/my.cnf
basedir = /application/mysql
datadir = /application/mysql/data

#加入开机自启
[root@db02 ~]# systemctl enable mysqld
Created symlink from /etc/systemd/system/multi-user.target.wants/mysqld.service to /usr/lib/systemd/system/mysqld.service.
```

## 升级安装5.7.24

##### 1.安装

```shell
#下载
[root@db01 ~]# wget https://downloads.mysql.com/archives/get/file/mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz

#解压
[root@db01 ~]# tar xf mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz

#安装
[root@db01 ~]# mv mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz  /application/mysql-5.7.24
```

##### 2.配置

```shell
#用户、启动脚本、配置文件都在5.6.44安装时拷过了，略过此步，直接授权并初始化

[root@db01 /application]# chown -R mysql.mysql /application/mysql*
[root@db01 /application/mysql]# ./bin/mysqld --initialize --user=mysql --basedir=/application/mysql-5.7.24 --datadir=/application/mysql-5.7.24/data
...
2019-11-4T08:50:24.896645Z 1 [Note] A temporary password is generated for root@localhost: 
kgMyi9%tHhjU
#初始化后看初始密码就成功了
[root@db01 /application]# echo $?
0
```

##### 3.启动

```shell
#二进制安装预编译安装在/usr/local下，需要改5.7.24的启停文件mysqld_safe
[root@db01 /application]# sed -i "s#/usr/local#/application#g" /application/mysql-5.7.24/bin/mysqld_safe

#停止旧版本mysql
[root@db01 /application]# systemctl stop mysqld

#启动新版本
[root@db01 /application]# /etc/init.d/mysqld start
```

##### 4.升级

```shell
#版本升级
[root@db01 /application]# rm -f mysql && ln -s mysql-5.7.24/ mysql
```

##### 5.systemd管理

```shell
[root@db02 ~]# vim /etc/my.cnf
basedir = /application/mysql
datadir = /application/mysql/data

[root@db01 /application]# systemctl start mysqld
[root@db01 /application]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since 一 2019-11-04 19:13:43 CST; 4s ago
   ...
```

##### 6.修改密码才能用

```shell
[root@db01 /application]# mysql -uroot -pkgMyi9%tHhjU
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.24

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> alter user root@'localhost' identified by '1';
Query OK, 0 rows affected (0.01 sec)
```

