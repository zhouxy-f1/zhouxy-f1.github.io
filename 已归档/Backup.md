## 实战案例：定时备份

##### 任务要求

```shell
#客户端
1.客户端提前准备存放的备份的目录，目录规则如下: /backup/web01_172.16.1.31_2018-09-02
2.客户端在本地打包备份(系统配置文件、应用配置等)拷贝至 /backup/web01_172.16.1.31_2018-09-02
3.客户端最后将备份的数据进行推送至备份服务器
4.客户端服务器本地保留最近7天的数据, 避免浪费磁盘空间
5.客户端每天凌晨1点定时执行该脚本

#服务端
1.服务端部署rsync，用于接收客户端推送过来的备份数据
2.服务端需要每天校验客户端推送过来的数据是否完整
3.服务端需要每天校验的结果通知给管理员
4.服务端仅保留6个月的备份数据,其余的全部删除
```

##### 建议备份的内容

```shell
1.系统配置文件
	/etc/rc.local						#开机自启配置文件
	/etc/fstab							#设备挂载配置文件
	/etc/hosts							#本地内网配置文件
2.应用程序或服务的配置文件
	/etc/nginx/nginx.conf		#nginx主配置文件
	/etc/nginx/conf.d/*			#nginx虚拟主机配置
	/etc/php.ini						#php主配置文件
	/etc/php-fpm.d/www.conf	#php进程配置文件
	/etc/redis							#redis配置文件
	/etc/my.cnf							#mysql配置文件
3.重要目录
	/var/spool/cron					#定时任务
	/etc/firewalld					#防火墙配置文件
	/server/scripts					#脚本目录
4.系统日志文件
	/var/log
```

##### 部署环境

| 主机名 | 外网IP    | 内网IP      | 角色   |
| :----- | :-------- | :---------- | :----- |
| backup | 10.0.0.41 | 172.16.1.41 | 服务端 |
| web01  | 10.0.0.7  | 172.16.1.7  | 客户端 |
| nfs01  | 10.0.0.31 | 172.16.1.31 | 客户端 |

## 操作步骤-客户端配置

**1.在家目录新建脚本文件：rsync.sh**

```shell
[root@web01 ~]# vim rsync.sh

#!/bin/bash
#定时任务不识别IP，需要加载环境变量
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
export PATH

#定义变量
H=`hostname`
IP=`ifconfig eth1|awk 'NR==2{print$2}'`
DATE=`date +%F`
SRC=${H}_${IP}_${DATE}

#在/backup/下新建存放备份的目录，以主机名_IP_日期命令
mkdir -p /backup/${H}_${IP}_${DATE}

#打包重要配置文件，放进刚建的目录
cd / && tar czf /backup/${SRC}/conf_${DATE}.tar.gz /etc/{rc.local,fstab,hosts,firewalld,passwd} /var/{log,spool/cron /server/scripts

#生成校验文件，放在/backup/下
md5sum /backup/${SRC}/conf_${DATE}.tar.gz > /backup/md5_${H}.txt:

#客户端只保留7天的备份数据
rm -fr $(find /backup -type d -mtime +6)

#同步时免密
export RSYNC_PASSWORD=123456
#同步/backup/目录
rsync -az /backup/  rsync_backup@172.16.1.41::zhouxy
```

**2.测试创建文件及打包脚本**

```shell
[root@web01 ~]# for n in `seq -w 30`;do date -s "201909$n";sh /root/rsync.sh;done
```

**3.写定时任务**

```shell
[root@web01 ~]#crontab -e
#每天凌晨1点执行备份脚本  By:zhouxy At:2019-8-2
00 01 * * * /bin/sh /root/rsync.sh &>/dev/null

#此时，定时任务的环境变量不能识别IP，需要在脚本里加载环境变量
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

## 操作步骤-服务端配置

**1.新建目录/backup**

```shell
[root@backup~]#mkdir /backup
```

**2.在家目录下写脚本文件：/root/rsync_md5.sh**

```shell
#!/bin/bash

PATH='/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin'
DATE=`date +%F`

#校验完整性并发邮件
md5sum -c /backup/md5*.txt |mail -s "${DATE}：校验结果" zhou012@yeah.net
#服务端仅保留6个月备份数据
rm -rf $(find /backup -type d -mtime +180)
```

**3.配置邮件服务**

```shell
#安装邮件服务
[root@backup~]# yum install -y mailx
#配置
[root@backup~]#vim /etc/mail.rc
文末添加下文：
set from=243074974@qq.com
set smtp=smtps://smtp.qq.com:465
set smtp-auth-user=243074974@qq.com
set smtp-auth-password=dhxpkaxdovwxbgee
set smtp-auth=login
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/
```

**4.写定时任务执行脚本**

```shell
#每天凌晨1点执行备份脚本  By:zhouxy At:2019-8-4
01 00 * * * /bin/sh /root/rsync_md5.sh &>/dev/null
```



