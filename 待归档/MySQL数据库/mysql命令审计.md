## mysql命令审计

参考Github：https://github.com/mcafee/mysql-audit/wiki/Installation

##### 第一步：安装插件

```bash
安装插件可以直接改配置文件重启mysql后安装，也可以在mysql命令行安装，但是涉及到THD偏移量时还是要改配置文件重启mysql，另外，安装之前要先关闭selinuxs


1.先下载插件并复制到mysql的插件目录再给执行权限
https://dl.bintray.com/mcafee/mysql-audit-plugin/						#下载插件的地址，需要下载跟mysql版本匹配的插件
cp audit-plugin-mysql-5.6-1.1.7-866/lib/libaudit_plugin.so /usr/lib64/mysql/plugin/
chmod +x /usr/lib64/mysql/plugin/libaudit_plugin.so 

2.计算THD偏移量
wget https://raw.githubusercontent.com/mcafee/mysql-audit/master/offset-extract/offset-extract.sh
chmod +x offset-extract.sh
yum install -y gdb
./offset-extract.sh  /usr/sbin/mysqld

3.安装插件
vim /etc/my.cnf
[mysqld]
plugin-load=AUDIT=libaudit_plugin.so
audit_offsets=7000, 7048, 4008, 4528, 72, 2704, 96, 0, 32, 104, 136, 7136, 4400, 2800, 2808, 2812, 536, 0, 0, 6368, 6392, 6376, 13056, 548, 516

/etc/init.d/mysqld restart
mysql -uroot -p
mysql> INSTALL PLUGIN AUDIT SONAME 'libaudit_plugin.so';
mysql> show plugins;
+----------------------------+----------+--------------------+--------------------+---------+
| AUDIT                      | ACTIVE   | AUDIT              | libaudit_plugin.so | GPL     |
+----------------------------+----------+--------------------+--------------------+---------+

```

##### 第二步：配置插件

```bash
可根据需要来配置：记录哪些命令，不记录哪些用户的命令等

mysql> show global variables like '%audit%'\G					#查看audit可配置参数
mysql> select @@audit_json_file;											#查看审计功能是否打开，1是打开0是未打开
mysql> set global audit_json_file=on;									#打开命令审计
mysql> select @@datadir;															#查看日志位置
mysql> select @@audit_json_log_file;									#查看日志文件名称
mysql> select @@audit_whitelist_users;								#查看白名单，即：不记录哪些用户的命令
mysql> select @@audit_record_cmds;										#查看审计哪些命令，默认是全部


由于global命令只对全局终端生效，但重启失效，所以需要把配置追加进配置文件
vim /etc/my.cnf
[mysqld]
audit_json_file=on
audit_record_cmds="insert,delete,update,create,drop,alter,grant,truncate"
audit_whitelist_users=wp
```

##### 第三步：使用插件

```bash
tailf /var/lib/mysql/mysql-audit.json
```

待解决问题：

1.日志文件有多大，需不需要做切割，切割策略怎样

2.json格式不方便查看，除了过滤日志有没有更好的方式