## 主从复制架构演变

```mysql
#基础架构
1主1从
1主多从
双主
多级主从
循环复制
MGR

#高可用架构
单活（需要切换）: MMM（KA+2主+1从） ，MHA（三节点，1主2从），TMHA(1主1从)
多活（不用切换）: NDB Cluster(官方收费)，InnoDB Cluster,PXC(Percona XtraDB Cluster),MGC(MariaDB Galera Cluster)

#高性能架构
读写分离：Atlas(360开发)，Cobar,ProxySQL(Percona开发)，MySQL Router(Oracle开发)，Maxscale,Mycat等
分布式架构：Atlas-Sharding(360在用)，Mycat(开源)，TDDL(淘宝在用)，Heisenberg(百度在用)，InnoDB Cluster
```



## MHA简介

```shell
#高可用（Master High Availability）
1.MHA是一个C/S结构的服务：由manager和node组成，manager是服务端，node是客户端
2.MHA可以安装在任意一台服务器上 
3.一个MHA管理节点可以管理上百套replication 
4.MHA管理节点，尽量避免装在主库上(避免断电，断网) 如果装在slave02上，slave02被提升为主库? 不让slave02提升为主库(no master) 

#优点
1)自动故障转移快 0-30秒即可完成
2)主库崩溃不存在数据一致性问题
3)不需要对mysql环境做重大修改
4)不需要添加额外的服务器，仅一台manager就可管理上百个replication
5)性能优秀，可工作在半同步复制和异步复制，当监控mysql状态时，仅需要每隔N秒向 master发送ping包(默认3秒)，所以对性能	无影响。你可以理解为MHA的性能和简单的主从复制框架性能一样。 icmp
6)只要replication支持的存储引擎，MHA都支持，不会局限于innodb
```

## MHA工作原理

当Master出现故障时，它可以自动将最新数据的Slave提升为新的Master,然后将所有其他的Slave重新指向新的Mastet

```shell

1）保存主库binlog日志，在/application/mysql/data下

2）选主
		当各个从库数据不一致时：优选数据量级最多的从库为备选主
		当从库数据一致时：依配置文件/etc/mha/tlbb.cnf中server的顺序选主
		如果server里设置了权重 candidate_master=1,则不论顺序，强制成为备选主
		默认slave落后master数据超100M，则该slave没有参与选主资格，配合用check_repl_delay=0解除默认限制
		
3）数据补偿
		当SSH能连上，从库对比主库GTID或position，用save_binary_logs工具截取主库binlog补偿到各从库
		当SSH不能连上时，用apply_diff_relay_logs工具识别从库relaylog间差异，多的补偿到少的4）Failover
		把旧主库踢除群
		将备选主切换为新Master，对外提供服务
		其它从库 change master to 新主库
5）应用透明
		连接mysql的应用还是指向旧主的IP，需要把VIP漂移到新主库，在不修改连接mysql程序代码情况下对外提供服务
		
6）故障通知 (send_repo)

7）二次数据补偿(binlog_server)
		另开一台服务器，实时保存主库binlog
		
8）修复宕掉的旧主库
		MHA是一次性切换主库，切换完自动退出程序，此功能暂未开发，需要写脚本实现
		
```

## MHA管理工具

```shell
#manager节点
[root@db01 ~]# ll mha4mysql-manager-0.56/bin/
-rwxr-xr-x 1 4984 users 2.0K 4月   1 2014 masterha_check_repl					#检查主从复制的状态
-rwxr-xr-x 1 4984 users 1.8K 4月   1 2014 masterha_check_ssh						#检查ssh key免密登录
-rwxr-xr-x 1 4984 users 1.9K 4月   1 2014 masterha_check_status				#检查MHA的启动状态
-rwxr-xr-x 1 4984 users 3.2K 4月   1 2014 masterha_conf_host						#删除故障节点
-rwxr-xr-x 1 4984 users 2.5K 4月   1 2014 masterha_manager							#启动MHA
-rwxr-xr-x 1 4984 users 2.2K 4月   1 2014 masterha_master_monitor			#监测主库心跳
-rwxr-xr-x 1 4984 users 2.4K 4月   1 2014 masterha_master_switch				#切换主库
-rwxr-xr-x 1 4984 users 5.1K 4月   1 2014 masterha_secondary_check			#建立TCP连接
-rwxr-xr-x 1 4984 users 1.7K 4月   1 2014 masterha_stop								#停止MHA

#node节点
[root@db01 ~]# ll mha4mysql-node-0.56/bin/
-rwxr-xr-x 1 4984 users  16K 4月   1 2014 apply_diff_relay_logs				#对比从库之间的relay log
-rwxr-xr-x 1 4984 users 4.7K 4月   1 2014 filter_mysqlbinlog						#截取binlog
-rwxr-xr-x 1 4984 users 8.1K 4月   1 2014 purge_relay_logs							#删除relay-log
-rwxr-xr-x 1 4984 users 7.4K 4月   1 2014 save_binary_logs							#保存所有binlog事件

```

## MHA部署

##### 1.准备工作

```shell
#主从复制
传统方法：
GTID方法：

#所有节点做软链接（因为程序原代码写死的指向/usr/bin/目录）
ln -s /application/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog 
ln -s /application/mysql/bin/mysql /usr/bin/mysql

#所有节点间配置互信（包括自己，以db01为例）
[root@db01 ~]# ssh-keygen
[root@db01 ~]# ssh-copy-id -i .ssh/id_rsa.pub  root@10.0.0.51
[root@db01 ~]# ssh-copy-id -i .ssh/id_rsa.pub  root@10.0.0.52
[root@db01 ~]# ssh-copy-id -i .ssh/id_rsa.pub  root@10.0.0.53
[root@db01 ~]# ssh-copy-id -i .ssh/id_rsa.pub  root@10.0.0.54
```

##### 2.安装软件包

```shell
#客户端安装到所有需要监控的节点（db01~04）
[root@db01 ~]# yum localinstall -y mha4mysql-node-0.56-0.el6.noarch.rpm
[root@db02 ~]# yum localinstall -y mha4mysql-node-0.56-0.el6.noarch.rpm
[root@db03 ~]# yum localinstall -y mha4mysql-node-0.56-0.el6.noarch.rpm
[root@db04 ~]# yum localinstall -y mha4mysql-node-0.56-0.el6.noarch.rpm

#服务端安装到任一从库（避免主库停电或断网无法提升从库为主库，多套架构可用单独一台服务器装mha服务端）
[root@db04 ~]# yum localinstall -y mha4mysql-manager-0.56-0.el6.noarch.rpm 
```

##### 3.配置服务端

```shell
#创建专用监控管理用户(只需要在主库上创建即可，其它从库自动复制)
mysql> grant all on *.* to mha@'10.0.0.%' identified by 'mha';

#配置文件
[root@db04 ~]# mkdir /etc/mha
[root@db04 ~]# mkdir /var/log/mha/tlbb -p
[root@db04 ~]# vim /etc/mha/tlbb.cnf
#通用配置
#指定日志名
#指定mha工作目录
#数据库binlog存放路径
#管理用户的用户名
#管理用户的密码
#2秒监测一下心跳（默认3秒）
#主从复制用户的用户名
#主从复制用户的密码
#做完互信的远程连接用户
candidate_master=1#强制成为备选主 
check_repl_delay=0#解除默认从库与旧主库数据差距超100M不参与选举的限制  
no master#已安装mha服务端的机器不参与选主

-------------- 配置文件开始点 ---------------

[server default]
manager_log=/etc/mha/manager.log
manager_workdir=/etc/mha/tlbb
master_binlog_dir=/application/mysql/data
user=mha
password=mha
ping_interval=2
repl_user=slave
repl_password=123
ssh_user=root

[server1]
hostname=10.0.0.51
port=3306

[server2]
hostname=10.0.0.52
port=3306
[server3]
hostname=10.0.0.53
port=3306
[server4]
hostname=10.0.0.54
port=3306


-------------- 配置文件结束点 ---------------

#强制选主使用场景
1.两地三中心
	比如三个机房分别在上海唐桥、上海青清、南京，主库目前在康桥，如果主库故障，首选切到青浦
2.keepalive高可用
	因为keepalive只支持两个节点之间切换，配置好高可用，必须指定备选主库
```

##### 4.启动前检查

```shell
# 检测ssh
[root@db04 ~]# masterha_check_ssh --conf=/etc/mha/tlbb.cnf
# 监测replication
[root@db04 ~]# masterha_check_repl --conf=/etc/mha/tlbb.cnf
```

##### 5.启动mha

```shell
#启动
[root@db04 ~]# nohup masterha_manager --conf=/etc/mha/tlbb.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /etc/mha/manager.log 2>&1 &

#检查
[root@db04 ~]# masterha_check_status --conf=/etc/mha/tlbb.cnf
tlbb (pid:18414) is running(0:PING_OK), master:10.0.0.51


#启动mha参数详解
nohup ： 											不挂断的运行，注意并没有后台运行的功能，，就是指，用nohup运行命令可以使命令永久的																执行下去，和用户终端没有关系，例如我们断开SSH连接都不会影响他的运行，注意了nohup															 没有后台运行的意思；

--remove_dead_master_conf      该参数代表当发生主从切换后，老的主库的ip将会从配置文件中移除。

--manger_log                   日志存放位置

-ignore_last_failover          在缺省情况下，如果MHA检测到连续发生宕机，且两次宕机间隔不足8小时的话，则不会进行																 Failover，之所以这样限制是为了避免ping-pong效应。该参数代表忽略上次MHA触发切换															 产生的文件， 默认情况下，MHA发生切换后会在日志目录下产app1.failover.complete															 文件，下次再次切换的时候如果发现该目录下存在该文件将不允许触发切换，除非在第一次切															 换后手动删除该文件，为了方便，这里设置为--ignore_last_failover
```

##### 6.宕机测试

```shell
#主库宕机
[root@db01 ~]# /etc/init.d/mysqld stop

#这时db02上没有slave
[db02]mysql> show slave status\G
Empty set (0.00 sec)
#db03上查看主库切到db02上了
[db03]mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.0.0.52
                  
#查看日志
[root@db04 ~]# tail -100 /etc/mha/manager.log 
```

## 修复旧主库

##### 修复步骤

```shell
1.修复损坏的旧主库(db01)
完全损坏的话要重新做：安装 -> 备份恢复 -> change master to  

2.db01指向新主
#mha服务端上查看指向新库的代码
[root@db04 ~]# grep -i 'change master to' /etc/mha/manager.log

#在旧主上执行，再启用
mysql> CHANGE MASTER TO MASTER_HOST='10.0.0.52', MASTER_PORT=3306, MASTER_AUTO_POSITION=1, MASTER_USER='slave', MASTER_PASSWORD='123';
mysql> start slave

3.db01加入集群
[root@db04 ~]# vim /etc/mah/tlbb.cnf
[server1]
hostname=10.0.0.51
port=3306

4.启动MHA并检查状态
[root@db04 ~]# nohup masterha_manager --conf=/etc/mha/tlbb.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /etc/mha/manager.log 2>&1 &
[root@db04 ~]# masterha_check_status --conf=/etc/mha/tlbb.cnf
```

##### 修复脚本

```shell

```

## 配置VIP漂移

##### 配置方法

```shell
1.修改配置文件（在配置文件指定VIP漂移脚本位置）
[root@db04 ~]# vim /etc/mha/tlbb.cnf
master_ip_failover_script=/etc/mha/master_ip_failover

2.完善配置

#上传脚本
见文末
#给脚本加执行权限
[root@db04 ~]# chmod +x /etc/mha/master_ip_failover

3.启动VIP漂移

#手动绑定VIP到新主库上（生成临时IP地址，网卡重启失效，如果要取消用 ifconfig eth0:1 down ）
[root@db04 ~]# ifconfig eth0:1 10.0.0.55/24

#最后重启mha
[root@db04 mha]# masterha_stop --conf=/etc/mha/tlbb.cnf
[root@db04 ~]# nohup masterha_manager --conf=/etc/mha/tlbb.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /etc/mha/manager.log 2>&1 &
[root@db04 ~]# masterha_check_status --conf=/etc/mha/tlbb.cnf

-------------------------- 补充 -------------------
#如果脚本起不来，原因可能是
1.权限问题：没有执行权限
2.语法问题
3.格式问题：脚本中有中文字符
解决格式问题：
[root@db04 /etc/mha]#  yum install -y dos2unix
[root@db04 /etc/mha]# dos2unix master_ip_failover                   
dos2unix: converting file master_ip_failover to Unix format ...

```

##### vip漂移脚本

```shell
[root@db04 /usr/local/bin]# vim master_ip_failover 
#!/usr/bin/env perl

use strict;
use warnings FATAL => 'all';
use Getopt::Long;
my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
#生产中需要修改（VIP地址需要跟公网同一网段，且是空闲地址）
my $vip = '10.0.0.55/24';
my $key = '1';
my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();
sub main {
    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
    if ( $command eq "stop" || $command eq "stopssh" ) {
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {

        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
sub stop_vip() {
     return 0  unless  ($ssh_user);
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
```

## 故障邮件提醒

##### 配置方法

```shell
1.修改配置文件
[root@db04 ~]# vim /etc/mha/tlbb.cnf 
report_script=/etc/mha/send

2.准备邮件脚本



3.重启MHA
[root@db04 ~]# nohup masterha_manager --conf=/etc/mha/tlbb.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /etc/mha/manager.log 2>&1 & 
[root@db04 ~]# masterha_check_status --conf=/etc/mha/tlbb.cnf
tlbb (pid:22813) is running(0:PING_OK), master:10.0.0.51

```

##### 脚本

```shell
#测试用

#!/bin/bash
/etc/mha/sendEmail -o tls=no -f zhou012@yeah.net  -t 243074974@qq.como  -s smtp.126.com:25 -xu zhou012@yeah.net -xp X!@oy0ng -u "MHA warning" -m "Your mah get failovered" &>/tmp/sendmail.log


#生产用

```

## 备份主库binlog

##### 配置方法

```shell
1.修改配置文件
#[root@db04 /etc/mha]# vim tlbb.cnf
[binlog1]
no_master=1
hostname=10.0.0.54
master_binlog_dir=/backup/binserver

2.完善配置
[root@db04 ~]# mkidr /backup/binserver -p
[root@db04 ~]# chown -R mysql.mysql /backup/*

3.从主库拉取binlog
[root@db04 ~]# cd /backup/binserver/
[root@db04 /backup/binserver]# mysqlbinlog -R --host=10.0.0.52 --user=mha --password=mha --raw  --stop-never mysql-bin.000002 &

4.重启mha
[root@db04 ~]# nohup masterha_manager --conf=/etc/mha/tlbb.cnf --remove_dead_master_conf -- ignore_last_failover < /dev/null > /etc/mha/manager.log 2>&1 & 
[root@db04 ~]# masterha_check_status --conf=/etc/mha/tlbb.cnf
tlbb (pid:22813) is running(0:PING_OK), master:10.0.0.52

#测试（在主库刷新binlog，在mha服务端查看备份目录）
[db02] mysql> flush logs;
Query OK, 0 rows affected (0.01 sec)

[root@db04 /backup/binserver]# ll
total 44K
-rw-rw---- 1 root root 380 Nov 25 19:23 mysql-bin.000002
-rw-rw---- 1 root root 238 Nov 25 19:23 mysql-bin.000003
-rw-rw---- 1 root root 191 Nov 25 19:23 mysql-bin.000004
-rw-rw---- 1 root root 238 Nov 25 19:18 mysql-bin.000005


#注意事项
1、用于备份主库binlog的目录不可跟本机binlog目录位置一样，否则本机binlog会错乱
2、备份日志应从复制主库日志最慢的从库开始
```

##### 从备份恢复数据到新主库

```shell

```



## 主从读写分离

##### atlas简介

```shell
 Atlas是由 Qihoo 360, Web平台部基础架构团队开发维护的一个基于MySQL协议的数据中间层项目。
它是在mysql-proxy 0.8.2版本的基础上，对其进行了优化，增加了一些新的功能特性。
360内部使用Atlas运行的mysql业务，每天承载的读写请求数达几十亿条。
下载地址
https://github.com/Qihoo360/Atlas/releases
注意：
1、Atlas只能安装运行在64位的系统上
2、Centos 5.X安装 Atlas-XX.el5.x86_64.rpm，Centos 6.X安装Atlas-XX.el6.x86_64.rpm。
3、后端mysql版本应大于5.1，建议使用Mysql 5.6以上


#Atlas主要功能
1.读写分离 
2.从库负载均衡
3.IP过滤
4.自动分表 
5.DBA可平滑上下线DB 
6.自动摘除宕机的DB

#Atlas相对于官方MySQL-Proxy的优势
1.将主流程中所有Lua代码用C重写，Lua仅用于管理接口 
2.重写网络模型、线程模型 
3.实现了真正意义上的连接池 
4.优化了锁机制，性能提高数十倍
```

##### atlas配置

```shell
#安装
[root@db04 ~]# rpm -ivh Atlas-2.2.1.el6.x86_64.rpm
[root@db04 /usr/local/mysql-proxy]# ll
total 0
drwxr-xr-x 2 root root  75 Nov 25 10:11 bin
drwxr-xr-x 2 root root  22 Nov 25 10:11 conf
drwxr-xr-x 3 root root 331 Nov 25 10:11 lib
drwxr-xr-x 2 root root   6 Dec 17  2014 log

#配置atlas
[root@db04 /usr/local/mysql-proxy/conf]# cp test.cnf  tlbb.cnf
[root@db04 /usr/local/mysql-proxy/conf]# vim tlbb.cnf 

[root@db04 /usr/local/mysql-proxy/bin]# ./encrypt 123
3yb5jEku5h4=


#启动atlas
[root@db04 /usr/local/mysql-proxy/conf]# /usr/local/mysql-proxy/bin/mysql-proxyd  tlbb start 
OK: MySQL-Proxy of tlbb is started

```

##### atlas管理

```shell
#从db04进入管理接口
[root@db04 ~]# mysql -uuser -ppwd -h127.0.0.1 -P2345

#管理命令
mysql> select * from help;
+----------------------------+---------------------------------------------------------+
| command                    | description                                             |
+----------------------------+---------------------------------------------------------+
| SELECT * FROM help         | 查看帮助
| select * from backends     | 查看后端的服务器状态
| set offline $backend_id    | 平滑下线 例如:set offline 2;
| set online $backend_id     | 平滑上线 例如:set online 2;
| add master $backend        | 添加后端主库:add master 10.0.0.55:3306;
| add slave $backend         | 添加后端从库:add slave 10.0.0.56:3306;
| remove backend $backend_id | 删除后端节点: remove backend 1;
| select * from clients      | 查看允许连接的客户端IP
| add client $client         | 添加客户端IP：add client 10.0.0.51;
| remove client $client      | 删除客户端IP：remove client 10.0.0.51
| select * from pwds         | 查看后端数据库的用户名和密码
| add pwd $pwd               | 添加用户，自动加密：add pwd root:123;
| add enpwd $pwd             | 添加用户，需要手动加密：add enpwd zhouxy:3yb5jEku5h4=;
| remove pwd $pwd            | 删除没有用的用户：remove pwd zhouxy;
| save config                | 保存到配置文件，不需要重启atlas
| seletc version             | 查看版本信息
+----------------------------+---------------------------------------------------------+


#当主库宕机后需要修改配置文件中主从库位置，不然主库变只读了，修改完直接save config，可保存至配置文件，不用重启atlas




#写脚本实现
sed -nr 's#^Master (.*)\(.*down)!#\1#gp' /etc/mha/manager.log
```

##### 脚本实现切换后（新主库写，旧主库读）

```shell

```



# MHA生产中上线方案

```shell
#区分操作是否需要停库
```

