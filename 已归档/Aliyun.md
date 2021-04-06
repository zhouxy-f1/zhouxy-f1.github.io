![image-20200308142641181](https://tva1.sinaimg.cn/large/00831rSTgy1gcrl6oxvovj30p308labf.jpg)

```bash
1.创建云服务器ECS
2.连接云数据库RDS，并实现读写分离
3.创建文件存储NAS
4.配置负载均衡LSB
5.使用Redis解决会话共享问题
6.ESS实现弹性伸缩
```

## 一、云服务器ECS

##### 创建实例

```bash
#付费模式
包年包月
按量付费
抢占式：实例价格根据供需关系波动，只要出价高于最低价即可，如果出价低于市场最低价则自动释放，无提醒，便宜但危险

#地域与可用区
地域：不同城市
可用区：该城市内不同机房，用不同的交换机连接，独立供电与网络，各区内网互通；相同区网络延时低，不同区容灾能力强

#实例分类
共享型：多人共用一台服务器，每人只能用最高20%的CPU资源
突发性能实例：允许短暂超过20%的cpu资源，超出部分另付费
```

##### 专有网络VPC

```bash
专有网络：加入逻辑隔离的私有网络，可以自定义网络拓扑和IP地址
经典网络：同区的公司内网互通，容易被攻击，不安全，已弃用

创建路由器和交换机必须跟ECS在同一个区，否则无法实用
```

##### 弹性公网EIP

```bash
公网IP：勾选则代表选择经典网络中的IP，直接绑死在这台主机上
EIP：建议选择弹性公网IP，还可换绑到其它ECS实例、SLB实例、NAT网关等
EIP在主机看不到公网IP
BGP多线: 路由器接入移动联通和电信3条专线，自动检测用户用的是移动联通还是电信，使用对应的专线，解决不同网络间的延时
```

##### 安全组

```bash
ECS的防火墙，是对路由器设置的出站入站规则
网络类型: 专有网络
```

#####远程连接ECS

```bash
#使用密钥对连接
1.Macbook上SecureCRT配置
Tools-Create public key-RSA-密码空-2048-OpenSSH Key format(new)-文件名改为.ssh2,
注意：提示使用此密钥为全局密钥时，一定要选否，不然Vmware连不了CRT
2.新建实例时导入密钥对
[root@macbook ~]# cat /Users/zhouxy/.ssh2/identity.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDX2Mb4SGu39w0t+UBhwxY4NguifbkPIKwQt6gmW1IZCVURAPRdUdekKJ4+qemAtrfw9wTTQAUp80cmt8Mxicec30LN1vdFRq4H1ui3+u+Y/us+e2Toy5g/WDhGs0SEgMLGWH56T3h944df/GUX0OP5NnG5CtpN1BKtsi32I2bD09JVcqdF/nfvqQc1J/N08ONWcklTV4ZKeCJqkPMvxp2YxW4dCjD9LLdo3xJKQ6E1OH7f2nmGcaenR7tu5PrVcqTxaIkZmZfbKOBzN9nbAZcl4hOMfbcIwiFAkXXEmTNJmaaETRGHCj1ua6kq0pERFuNBPj2kxTAb2mdN+2HuBqXJ zhouxy@macbook

#通过密码连接
新建实例时选择密码连接，或新建完实例在"更多"里选择重置实例密码，再重启即可
```

##### 增删云盘

```bash
实例-存储与快照-云盘
#查看
[root@web data]# df -hT								//查看磁盘详情（大小、使用率、挂载点等）
[root@web ~]# lsblk										//列出所有块设备，查看新加磁盘是否被识别

#分区-格式化-挂载-生效
[root@web ~]# fdisk /dev/vdb					//n新建分区 w保存 p查看
[root@web ~]# mkfs.xfs /dev/vdb 			//格式化
[root@web ~]# mount /dev/vdb /data		//临时挂载
[root@web ~]# cat /etc/fstab 					//永久挂载
/dev/vdb              /data        xfs        defaults        0 0	（注：不用dump备份，不用fsck检查）
[root@web ~]# mount -a								//不重启生效

#释放云盘（linux卸载-云盘卸载-释放）
[root@web ~]# umount -l /data					//-l --lazy 断开文件系统再卸载
```

##### 其它管理

```bash
#救援模式
实例-远程连接-reboot

#修改私网IP
实例-详情-配置信息-修改私网IP地址(需要关机状态下操作)

#监控
实例详情-监控信息
```

##### 部署wordpress

```bash
#安装配置nginx
yum install -y nginx
sed -i '37,87d' /etc/nginx/nginx.conf
cat >/etc/nginx/conf.d/blog.conf<<EOF
server {
        listen 80;
        server_name localhost;
        root /code/wordpress;
        index index.php index.html;
 
        location ~ \.php$ {
                root /code/wordpress;
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
                }
        }
EOF
nginx -t
systemctl start nginx
netstat -lntup|grep 80

#安装部署php
cd /opt
tar xf php71.tar.gz
yum localinstall *.rpm -y
sed -i 's#user = apache#user = nginx#g' /etc/php-fpm.d/www.conf
sed -i 's#group = apache#group = nginx#g' /etc/php-fpm.d/www.conf
systemctl start php-fpm
netstat -lntup|grep php
ps -ef|grep php

#部署代码
yum install unzip -y
cd /opt/
mkdir /code/
unzip wordpress-5.3.2.zip -d /code/
chown -R nginx:nginx /code/
curl -I 127.0.0.1:8080
```

## 二、云数据库RDS

##### 创建实例

```bash
#类型
mysql版本要跟要迁移的数据库保持一致
高可用版
可用区要跟ECS所在区一致，如果不一致则要新建一个switch，通过内网能连接，但延时高
```

##### 内网连接数据库

```bash
#获取内网地址
实例详情-设置白名单-切换高安全白名单模式-添加白名单分组-加载ECS内网IP（更安全，但后期增加ECS还需要添加进白名单）
或者：直接修改default专有网络组中组内白名单为：172.16.1.0/24

#新建数据库
数据库管理-名称：wordpress-创建

#创建账号
账号管理-普通账号-授权为读写

#测试连接
[root@web01 ~]# yum install -y mariadb
[root@web ~]# mysql -hrm-uf6xh79a418ft661y.mysql.rds.aliyuncs.com -uwp -padmin-123
```

##### 外网连接数据库

```bash
需要把外网连接数据库的IP加入白名单
RDS-数据安全性-添加白名单分组-隔离模式：经典网络及外网地址-添加ip：223.167.68.159-完成

#通过Navicat测试
Host:rm-uf6xh79a418ft661yro.mysql.rds.aliyuncs.com
Port:3306
Usrname:wp
Password:admin-123
连接成功！
```

##### 数据库读写分离

```bash
#配置读写分离
1.添加只读实例
实例详情-申请读写分离地址-开启数据库代理服务：代理个数为1（等5分钟可开通好）-添加只读实例-购买
2.申请读写分离地址
开启读写分离连接-延迟阀值：30秒；读权重分配：主实例0，只读实例100（0代表不分配读请求，只读实例按权重比例分配读请求）
得到读写分离地址：hvtv1pmk7cvjl9gx2d5k-rw4rm.rwlb.rds.aliyuncs.com

#使用读写分离
应用程序----连接读写分离地址----自动分配读请求到只读实例，写请求到主实例
```

##### 数据库备份

```bash
#手动备份
实例详情-右上角备份实例-在左栏备份恢复中查得结果

#自动备份
实例管理-备份恢复-备份设置
```

## 三、共享存储NAS

##### 文件存储 NAS

```bash
1.NAS文件系统-文件系统列表-创建文件系统
2.文件系统列表-更多-添加挂载点（哪个网络可以用这个nfs）
3.购买存储包-绑定
4.挂载文件系统到ECS（文件系统列表-详情-挂载使用）
		先安装nfs: yum install nfs-utils
		再挂载:
mount -t nfs -o vers=3,proto=tcp,nolock,noresvport 033e3f0d-odiy.cn-shanghai.extreme.nas.aliyuncs.com:/share /code/wordpress/wp-content/uploads
		最后加入自启：vim /etc/fstab
UUID=033e3f0d-odiy.cn-shanghai.extreme.nas.aliyuncs.com:/share /code/wordpress/wp-content/uploads  nfs defaults 0 0
```

## 四、负载均衡SLB

##### 新建实例并配置

```bash
#新建实例
负载均衡SLB-实例管理-创建负载均衡
可用区域: 阿里SLB自带高可用，主备不同区，主承担主要流量，主不可用时承担流量
实例规则: 根据连接数来选择，后期可升级

#配置
负载均衡SLB-实例列表-点我开始配置
协议&监听: HTTP协议，端口80
调度算法：轮询
新建虚拟服务器组：webs
添加服务器：web01,web02，端口80

#健康检查两种方式
1.基于TCP: 检查端口，只要端口开放，就认为web服务器存活
2.基于http，检查路径或域名，不仅要端口开放，还要能回复页面
```

##### 负载相关补充

```bash
#开启负载必然带来会话问题，解决办法如下：
1.基于IP：访问IP首次访问哪台web，下次请求还分配到该web，若此IP为大型局域网公共出口，则该web压力过大
2.基于cookie：首次访问web时下发sessionid，保存到浏览器cookie，下次请求对比session id，lb分配到上次web
3.最常用的是把会话信息保存到redis里，或nfs里的文件


#私网负载均衡使用场景
1)数据库读写分离，主写从读，从库接四层负载，轮询读从库
2)web服务器需要调用应用程序时，应用处理服务器前接入内网负载，web连负载的IP即可
3)接redis集群

#虚拟服务器组作用？
把不同的业务转发到不同的后端服务器
web1 aaa.com bbb.com
web2 aaa.com

aaa.com
upstream web_site{
	server web1:80;
	server web2:80;
}
bbb.com
upstream web_site{
	server web1:80;
}
```

##### 配置四层负载

```bash
#端口转发
后台web组没有公网时，配置四层负载做端口转发即可

负载均衡-实例管理-监听配置向导-TCP协议-监听端口xxxx-新建虚拟服务器组-添加服务器-服务器端口22
```

##### https

```bash
阿里的证书只需要在SLB上配置，不需要在web上配置443端口，开启80就行了

#申请证书，部署到SLB即可
SSL证书-购买证书-【品牌：symantec 类型：免费版（个人）】-提交
购买成功后：申请证书-状态变成已签发后-部署到云产品-选择SLB
```

## 五、数据库REDIS

##### 创建实例

```bash
版本类型：社区版
版本号：redis 4.0
架构类型：标准版
节点类型：双副本
实例规格：1G主从版
```

#####  部署dz论坛

```bash
#nginx添加主机
[root@web conf.d]# cat dz.conf 
server {
 listen 8090;
 server_name localhost;
 root /code/dz;
 index index.php index.html;
 location ~ \.php$ {
 root /code/dz;
 fastcgi_pass 127.0.0.1:9000;
 fastcgi_index index.php;
 fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
 include fastcgi_params;
 }
}

#mysql中添加dz库
RDS列表-管理-数据库管理-账号管理

#网页配置
输入：http://101.132.27.208:8090/
```

##### dz论坛连redis

```bash
#redis配置
实例信息-开启免密访问

#测试redis连接
yum install redis -y
redis-cli -h r-uf6760sfu9ou8s48p9.redis.rds.aliyuncs.com set k1 v1
OK

#配置dz站点
[root@web ~]# vim /code/dz/config/config_global.php
$_config['memory']['redis']['server'] = 'r-uf6xzfpqcrdp1g1c5k.redis.rds.aliyuncs.com';

#检查dz站点
管理中心-全局-性能优化-内存优化
此时：redis项config设置已打开

也可以浏览一些页面，再keys *查看缓存数据量（慎用）
```

##### 解决会话共享

```bash



```

## 六.快照与镜像

##### 快照与镜像

```bash
#作用
快照跟vmware的快照性质一样，可基于快照制作一个包含所有已安装软件的镜像，从而快速复刻一台ECS主机
注意事项：
做快照做消耗IO，避免在业务进行时做快照，快照前最好先删不必要的文件再快照，另外关机才能回滚

#制作快照
先购买预付费存储包

1.自动快照
ECS实例-存储与快照-快照-创建策略-设备磁盘

2手动快照（约需4分钟）
ECS实例-实例-详情-本实例磁盘-创建快照
ECS实例-存储与快照-云盘-创建快照

#打包镜像
ECS实例-存储与快照-快照-创建自定义镜像
埴写：名称、描述、默认资源组

共享镜像
适用于把A账户的实例迁移到B账户，具体做法如下：
A账户做快照-打包成镜像-分享给B账户-B账户新建实例时镜像先共享镜像

导入镜像
云服务器ECS-快照和镜像-镜像-导入镜像
可将KVM主机导入为镜像
```

##### 快速复刻web02

```bash
#基于镜像创建云主机
ECS-实例列表-更多-购买相同配置
```

## 七、弹性伸缩ECS

```bash
1.创建一台ECS云主机
2.准备SLB负载均衡，通过SLB的TCP代理至后端的ECS的22端口
3.在ECS云主机上安装nginx+php
4.配置nginx+php+代码，启动服务，并加入开机自启
5.接入负载，将所有请求调度到后端web节点
6.将真实的公网域名解析到负载均衡
7.配置RDS数据库，包括内网地址、数据库和账号等
8.实现自动监控，弹性伸缩ECS云主机
	1）把已经配置软件和服务ECS做好快照并打包成镜像
	2）创建弹性伸缩组，组内实例来源选自定义，在增加伸缩配置中选刚打包好的镜像，再选好负载均衡和数据库等
	3）点击弹性伸缩组-伸缩规则，配置添加和减少ECS的规则
	4）点击弹性伸缩服务-自动触发任务管理-报警任务
```

## 八、释放所有服务器

```bash
1.在控制台左上角，资源管理可查看所有资源，可查看还落下什么没有释放的
2.释放ESC、RDS（先关闭读写分离）、redis、LSB实例-安全组-文件存储-再依次删交换机、专有网络、弹性公网IP等-再依次删除镜像、快照等
3.关闭读写分离：RDS主库详情-数据库代理-读写分离-关闭读写分离
```

