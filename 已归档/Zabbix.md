## 	zabbix安装配置

##### 安装zabbix服务端

```bash
zabbix实际部署在LAMP环境中，zabbix-web-mysql中集成了apache和php，mysql数据库一般需要一台独立的主机

#1.安装软件
rpm -ivh http://repo.zabbix.com/zabbix/4.0/rhel/7/x86_64/zabbix-release-4.0-1.el7.noarch.rpm
yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent mariadb-server
#2.mysql建用户建库,并导入zabbix表
systemctl start mariadb											
mysql
create database zabbix character set utf8 collate utf8_bin;
grant all privileges on zabbix.* to zabbix@'localhost' identified by 'zabbix';
quit;
zcat /usr/share/doc/zabbix-server-mysql-4.0.19/create.sql.gz  |mysql -uroot zabbix
#3.配置zabbix-server连接数据库，并启动zabbix-server
vim /etc/zabbix/zabbix_server.conf						
第91行：DBHost=localhost
第125行：DBPassword=zabbix
systemctl restart zabbix-server
systemctl enable zabbix-server
#4.修改zabbix-web前端时区
vim /etc/httpd/conf.d/zabbix.conf							
第19行：php_value date.timezone Asia/Shanghai
systemctl start httpd
systemctl enable httpd

#5.打开网址
http://47.101.205.197/zabbix/					默认用户名密码：Admin  zabbix

常见问题
1.如果检查php配置问题就修改/etc/php.ini文件，再重启httpd服务即可。配置完成后，生成配置文件，位于：/etc/zabbix/web/zabbix.conf.php

2.页面上有中文乱码问题
yum install wqy-microhei-fonts -y
cp /usr/share/fonts/wqy-microhei/wqy-microhei.ttc /usr/share/zabbix/assets/fonts/graphfont.ttf
```

##### 添加客户端

```bash
zabbix可用于监控交换机、路由器、防火墙用SNMP接口；监控tomcat用JMX接口；监控硬件用IPMI接口，监控系统及应用用代理接口
每增加一个新被控端，zabbix-server的数据库容量就需要多100M。按保留90天数据、500监控项、1事件算，每监控一台主机至少需要100M磁盘空间，其中历史数据57M，趋势4M,事件30M

#在被控端安装zabbix-agent
yum install -y zabbix-agent 
vim /etc/zabbix/zabbix_agentd.conf
第98行：Server=10.0.20.88
systemctl start zabbix-agent

#在zabbix-server页面添加主机
添加：点击配置-主机-创建主机（名称写主机名，agent代理程序的接口：node1的ip）-添加模板-添加
查看：监测-最新数据-选择主机-应用 或者 监测-图形-选择群组、主机、图形-应用
```

## 自定义监控项

```bash
#添加自定义监控项
1.先在node1上添加监控项目
vim /etc/zabbix/zabbix_agentd.d/tcp_state.conf 
UserParameter=tcp_state[*],netstat -ant|grep -c $1
检查取值测试
在node1上：zabbix_agentd -p|grep state[LISTEN]
在server端：zabbix_get -s 10.0.20.90 -k tcp_state[LISTEN] 				//yum install -y zabbix-get

2.再去sever界面添加监控项
配置-主机-web01的监控项-创建监控项-（名称：可用中文，键值必须是login_num）-添加
检查能否取到新数据
监测-最新数据-过滤web组、应用集，查看是否取到值

#添加触发器
配置-主机-node1的触发器-创建触发器-（名称:可用中文；表达式：添加-选监控项-表达式；要写描述）-添加
需要多条件成立才触发时：在表达式下面点添加-选择同时满足
需要恢复时也触发某动作时：写恢复表达式

⇒ avg(#5) → 最近5个值的平均值
⇒ avg(1h) → average value for an hour
⇒ avg(1h,1d) → average value for an hour one day ago.
⇒ count(10m) 						→ 10分钟内取到值的数量
⇒ count(10m,"error",eq) → 10分钟内取的值中'error'有多少个
⇒ count(10m,12) 				→ 10分钟内取到的值中12有多少个
⇒ count(10m,12,gt) 			→ 10分钟内取到的值中大于12的有多少
⇒ count(#10,12,gt) 			→ 最新取到的10个值中，大于12的有多少
⇒ count(10m,12,gt,1d) 	→ 近10分钟内取到的大于12的值，比1天之前的
number of values for preceding 10 minutes up to 24 hours ago that were over '12'
⇒ count(10m,6/7,band) → number of values for last 10 minutes having '110' (in binary) in the 3 least significant bits.
⇒ count(10m,,,1d) → number of values for preceding 10 minutes up to 24 hours ago

nodata(5m)表示最近5分钟得到的值
nodata(#5)表示最近5次得到的值
diff()		表示对比上一次文件的内容
last()		表示对比最新的值

#触发器依赖关系
假如监控了一个路由器和后面连的很多web，一旦路由器坏了，所有web也跟着报警，可配置web依赖路由器
点击配置-主机-node1的触发器-选择触发器-依赖关系-选择依赖的触发器

#触发后的动作(邮件通知)
1.先配置动作：点击配置-动作-事件源（触发器）-创建动作-操作（修改告警邮件内容）-恢复操作（也要改）-添加
修改告警邮件内容
故障{TRIGGER.STATUS},服务器:{HOSTNAME1}发生: {TRIGGER.NAME}故障!
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}
当前状态:{TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID:{EVENT.ID}　
-------------------------
恢复{TRIGGER.STATUS}, 服务器:{HOSTNAME1}: {TRIGGER.NAME}已恢复!
告警主机:{HOSTNAME1}
告警时间:{EVENT.DATE} {EVENT.TIME}
告警等级:{TRIGGER.SEVERITY}
告警信息: {TRIGGER.NAME}
告警项目:{TRIGGER.KEY1}
问题详情:{ITEM.NAME}:{ITEM.VALUE}
当前状态:{TRIGGER.STATUS}:{ITEM.VALUE1}
事件ID:{EVENT.ID}
2.再配置报警媒介：点击管理-报警媒介类型-Email(SMTP服务器：smtp.qq.cm；smtp端口：465；SMTP HELO：qq.com；SMTP电邮：243074974@qq.com；安全链接：SSL；勾选SSL验证对端 验证主机；认证：用户名：243074974@qq.com，密码是授权码rgrgsxvzcehgbjeh)
3.最后配置收件箱：右上角小人-报警媒介-添加-Email(zhou012@yeah.net)-更新
如果邮箱配置存在错误，单击报表->动作⽇志->检查邮箱发送情况

#梯度报警
警报到先运维组，30分钟未解决的话警报就发给经理组，经理20分钟未解决就把邮件发给总监
先添加用户组和用户：点击管理-用户群组-创建用户群组-用户-创建用户
再到动作里添加操作步骤：点击配置-动作-选择Report**-操作-新的(步骤：1-1表示告警1次；持续时间：20m 表示这2个步骤共持续20分钟，默认不写则是1小时)-添加-新的（步骤3-4...）-更新
解释：出现警报立即发给运维组，20分钟后发邮件到经理组，再10分钟发邮件到总监组。

#添加图形
添加：点击配置-主机-node1的图形-创建图形-（监控项选同应用集的）-添加
查看：点击监测-图形-选择群组和主机-图形选tcp_state即可

#聚全图形
添加：点击监测-聚合图形-创建聚合图形

#幻灯片演示
添加：点击监测-聚合图形-选择幻灯片演示-创建幻灯片播放
查看：点击监测-聚合图形-选择幻灯片演示-点击已建好的幻灯片即可。注意配置延迟时间，默认30秒

#模板
模板里包含应用集、监控项、触发器、图形、聚合图形、自动发现、Web监测等，创建模板后只要改模板，关连的主机全会改掉，模板也支持导入导出。但是新机器还需要把.conf配置文件和取值脚本都拷过来，才能通过模板取到值

点击配置-模板-创建模板-找到相关主机的监控项-复制-（类型：模板；目标：刚建的模板）-复制
点击配置-模板-导入或导出，
```

## 监控应用程序

```bash
#监控nginx
1.先打开nginx状态检查
less /etc/nginx/conf.d/zabbix.conf
...
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
        access_log off;
    }
...

2.配置监控项
less /etc/zabbix/zabbix_agentd.conf
# Web server checks
UserParameter=nginx[*], 					bash /home/zabbix/agent_bin/nc_nginx_check.sh $1
UserParameter=get_nginx_error[*], bash /home/zabbix/agent_bin/nc_nginx_error_check.sh $1 $2

3.写取值脚本
less /home/zabbix/agent_bin/nc_nginx_check.sh
#!/bin/bash
case $ZBX_REQ_DATA in
  [aA]ctive_connections)    echo "$NGINX_STATS" | head -1             | cut -f3 -d' ';;
  [aA]ccepts)               echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f2 -d' ';;
  [hH]andled)               echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f3 -d' ';;
  [rR]equests)              echo "$NGINX_STATS" | grep -Ev '[a-zA-Z]' | cut -f4 -d' ';;
  [rR]eading)               echo "$NGINX_STATS" | tail -1             | cut -f2 -d' ';;
  [wW]riting)               echo "$NGINX_STATS" | tail -1             | cut -f4 -d' ';;
  [wW]aiting)               echo "$NGINX_STATS" | tail -1             | cut -f6 -d' ';;
  [vV]ersion)               echo "$NGINX_VERSION";;
  *) echo $ERROR_WRONG_PARAM; exit 1;;
esac

#监控php
vim /etc/php-fpm.d/www.conf
pm.status_path = /phpfpm_status

   location ~ ^/(phpfpm_status)$ {
        include fastcgi_params;
        fastcgi_pass    127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}

[root@Agent ~]# curl http://127.0.0.1/phpfpm_status
pool:                 www														fpm池名称,大多数为www
process manager:      dynamic												进程管理方式dynamic或者static
start time:           05/Jul/2016:15:30:56 +0800		启动日志,如果reload了fpm，时间会更新
start since:          409														运行时间
accepted conn:        22														当前池接受的请求数
listen queue:         0															请求等待队列,如果这个值不为0,那么需要增加FPM的进程数量
max listen queue:     0															请求等待队列最高的数量
listen queue len:     128														socket等待队列长度
idle processes:       4															空闲进程数量
active processes:     1															活跃进程数量
total processes:      5															总进程数量
max active processes: 2															最大的活跃进程数量（FPM启动开始计算）
max children reached: 0															进程最大数量限制的次数，如果不为0，可调大最大进程数


#监控mysql
安装Percona Monitoring Plugins
yum install -y http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
yum install percona-zabbix-templates -y

```

## 监控web网站

```bash


```

## 自动化监控

```bash
#自动化监控
被动模式：又称自动发现，是由server端按ip段轮询扫描agent端，而且需要server不停地向agent要监控项的值
主动模式：又称自动注册，是由agent端主动加入server监控，取值是由agent端收集好一次性发给server端

1.先把所有取值脚本和.conf配置文件推送到新机器，取值脚本位置在.conf文件里指定
2.准备主动式的模板（如果不改也能取到值，但还是由server向agent要数据）
点击配置-模板-选择模板-全克隆-（改名：Template* Active）-找到刚改名模板的监控项-全选监控项-批量更新-类型改为主动式
3.在server上配置自动注册后的动作（根据不同的主机名来匹配不同的模板）
点击配置-动作-事件源选自动注册-创建动作-动作(名称：web自动注册；触发条件：主机名包含node) - 操作（新的；操作类型：依次添加主机、添加到组、与模板关联）-添加
4.最后把agent端改为主动模式
vim /etc/zabbix/zabbix_agentd.conf
ServerActive=10.0.20.88		启用主动模式
Server=10.0.20.88					被动模式的可以留着，不影响主动式
Hostname=zbx_node1				server端识别agent是以这个为准，页面显示的主机名也是这个，不指定则默认用系统主机名


问题：需要配置hostname，不能一台台改，如果批量操作
```

## 分布式监控

```bash
#分布式监控
当机器数量过千台或位于不同机房或网络不稳定时，需要用zabbix-proxy做代理，让agent自动注册到proxy，由proxy接收监控数据存入自己的数据库，然后统一发送给server，proxy不能发送警报、不能执行远程命令、不能处理事件，这些只能由server来处理
ProxyLocalBuffer=0		表示数据传递给server之后还要在proxy里保存多久（单位为小时）。如果注释就是代表不删除。
ProxyOfflineBuffer=1  表示数据没有传递给server的话还要在proxy里保存多久（单位为小时）。如果注释就是代表不删除。

yum install -y mariadb-server
systemctl start mariadb
mysql
create database zabbix_proxy default charset utf8;
grant all on zabbix_proxy.* to zabbix_proxy@'localhost' identified by 'zabbix_proxy';
exit
zcat /usr/share/doc/zabbix-proxy-mysql-4.0.19/schema.sql.gz |mysql -uzabbix_proxy -p zabbix_proxy

vim /etc/zabbix/zabbix_proxy.conf
Server=47.101.205.197
Hostname=hz_proxy
DBName=zabbix_proxy
DBUser=zabbix_proxy
DBPassword=zabbix_proxy
systemctl start zabbix-proxy
systemctl enable zabbix-proxy

去server端添加proxy
点击管理-agent代理程序-创建代理-（名称：必须是hz_proxy；代理程序模式：主动，意思是让proxy找server）-添加

配置自动注册（根据不同的主机名，匹配不同的模板）
点击配置-动作-事件源选自动注册-创建动作

修改所有agent端的Server和ServerActive指向proxy
vim /etc/zabbix/zabbix-agent.conf
Server=10.0.20.89
ServerActive=10.0.20.89
systemctl restart zabbix-agent
```

## zabbix优化

```bash
#当zabbix-server端很繁忙时，增加相应的进程数量对优化效果非常明显，当然进程数越多，占用的内存就越多
1.先查哪个进程卡：点击检测-图形-（群组：server；主机：server；图形：process busy %）
2.去server配置文件查对应的进程，适当调大点进程数
vim /etc/zabbix/zabbix-server.conf
StartDiscorvers=20
StartPollers=5

#当zabbix-server缓存不够时，适当增大缓存,查cache usage,% used表
vim /etc/zabbix/zabbix-server.conf
CacheSize=2G 			一般调到2-8G

#队列：原本该在指定时间内传送监控数据过来，却没送到，基本是线程繁忙，增加线程，若无效就要扩展硬件性能
右上角选细节，可查到是哪台主机，哪些监控项有延时
```

## API接口调用

```bash
#生成token(每次生成的都不同,这个令牌有用于操控该网站资源)
curl -s -X POST -H 'Content-Type:application/json' -d '
{
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
        "user": "Admin",
        "password": "zabbix"
    },
    "id": 1,
    "auth": null
}' http://10.0.20.88/zabbix/api_jsonrpc.php

执行结果：
{"jsonrpc":"2.0","result":"f790189dd705c9d18ef5adf70d72b0af","id":1}


/usr/share/zabbix


#使用令牌创建一台主机
curl -s -X POST -H 'Content-Type:application/json' -d '
{
    "jsonrpc": "2.0",
    "method": "host.create",
    "params": {
        "host": "zbx-node3",
        "interfaces": [
            {
                "type": 1,
                "main": 1,
                "useip": 1,
                "ip": "192.168.3.1",
                "dns": "",
                "port": "10050"
            }
        ],
        "groups": [
            {
                "groupid": "15"
            }
        ],
        "templates": [
            {
                "templateid": "10001"
            }
        ],
        "inventory_mode": 0,
        "inventory": {
            "macaddress_a": "01234",
            "macaddress_b": "56768"
        }
    },
    "auth": "f790189dd705c9d18ef5adf70d72b0af",
    "id": 1
}' http://10.0.20.88/zabbix/api_jsonrpc.php


zabbix_sender -z 172.16.1.71(服务端数据) -p 10051(服务端口) -s mysql02(本机名称) -k zabbix-sender (键值与刚刚写的键值一致)-o hello(  传输的数据)
```

```bash
crontab -lu ncbackup
cd /opt/backup/
cd file/
ps -ef|grep zabbix
grep zabbix /opt/ncscripts/cmd_track.log |grep restart
 crontab -l
date +'%F %T' -d @1587238201
cd file/
less /opt/ncscripts/backup/conf/srv-omp-ali-nfs2.conf

```





