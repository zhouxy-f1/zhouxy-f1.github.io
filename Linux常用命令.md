## 第一章 网络管理

##### DNS解析

```bash
协议://主机名.二级域名.顶级域名/资源记录

#DNS解析
选择域名服务器：优先识别eth0的配置，再读/etc/resolv.conf
本地域名解析：/etc/hosts

域名:baidu.com
主机名:baidu.com域内有很多主机，如www、zhidao、mail等
限定域名：www.baidu.com和zhidao.baidu.com等被称为FQDN(full qualified domain name)
顶级域名：TLD(top level domain),又叫一级域名，分为组织域如.com .org .net .cc，国家域如.cn .hk .jp 和反向域等
反向域名：提供ip地址到域名的对应关系，如X.X.X.in-addr.arpa
二级域名：baidu.com zlwm115.com等
资源记录：RR(resource record),DNS服务器的数据库中，每个条目都称作一个资源记录
超始授权记录：SOA(start of authority)，用来表明一个区域内部主从服务器如果同步数据及起始授权对象是谁的
DNS服务器：NS(name server)

#dig命令
yum install -y bind-utils				#下载命令，此软件包还包含nslookup等
dig @223.5.5.5 qq.com cname			#使用223的dns服务器，不指定的话默认用/etc/resolv.conf中的配置
dig -b 172.16.1.186 qq.com 			#-b指当本机有多个ip，指定哪个ip来向dns服务器发起查询请求，默认
dig qq.com a +tcp								#默认用udp发起查询请求 +tcp指定用tcp发请求
dig -x 223.5.5.5 +short					#-x反向解析ip +short指精简显示  可用+trace来追踪迭代查询的过程
dig -f /opt/domain.list 				#批量解析域名
dig qq.com soa +short						#查soa信息，该结果依次是：服务器 邮箱 序列号 3600 300 86400 缓存5分钟
ns1.qq.com. webmaster.qq.com. 1330914143 3600 300 86400 300 

#nslookup命令
1.非交互式查询
nslookup www.baidu.com								#查A记录和cname。Non-authoritative answer表示结果是从缓存拿的数据
nslookup -qt=ns baidu.com 						#查看一个域名对应的多个DNS服务器 -qt=type指定查询类型
nslookup -debug www.baidu.com					#查看DNS缓存保存时间TTL
nslookup zhidao.baidu.com 223.5.5.5		#223指定使用的DNS服务器，不指定则用/etc/resolv.conf中配置的DNS服务器

2.交互式查询
nslookup
>set type=ns
>baidu.com
>server 223.5.5.5
>www.baidu.com
>exit
```

##### 网络连通

```bash
#ip netstat tcpdump sar
ping 
tcping 10.0.0.100 22  #如果设置了禁拼
nethogs -v 3
可以查到是哪个进程占用带宽资源

#ss
ss是Socket Statistics缩写，用于获取socket统计信息
ss -ltuxp			#-t指显示tcp协议的sockets，-u是udp的，-x是unix的,-p显示处于监听端口进程 -l显示处于监听状态的端口
ss -

dig
#curl
curl -u user:pwd  http://man.linuxde.net		#完成http或ftp认证
curl -I www.baidu.com												#仅响应头部信息
curl -e www.baidu.com www.zlwm1152.com			#指定来源网站

#压测

#禁ping
echo '0或1' > /proc/sys/net/ipv4/icmp_echo_ignore_all				#1表示禁止别人ping本机，0代表允许
```

##### 系统时间

```shell
#定时关机
sync												#把内存中的数据写到磁盘中（关机、重启都需要先执行sync）
shutdown -h +10或13:00或0		 #定时关机，halt 10分钟后关机，13点整关机，立刻关机
shutdown -c  								#cancel取消关机预定设置
poweroff										#关机

#日历
cal 												#显示本月日历
cal 2019										#显示2019年全年的日历
cal 9 2020									#显示2019年9月的日历
cal -3											#显示上月本月和下月的日历
cal -y											#显示一整年的日历
cal -j -y										#以天数来显示日历，从1月1日到日历中所有日期的天数

#日期
date +%F 										#显示年月日 等价于 Ymd
date +%T	--time  					#显示时分秒  HMS
date +%y-%m-%d							#显示年份后两位-月份-日期
date "+%F +%T"							#显示当前时间，格式是：2020-08-22 21:12:31
date -d '-1day' +%F --dete  #打印当前系统时间减1天的结果，不改系统时间
date -s "19880214"  --set 	#修改当前系统日期为88年2月14日
date -s "12:00:00"					#修改当前系统时间为12点整
ntpdate time1.aliyun.com		#同步时间
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm	#centos8安装ntpdate
yum install -y wntp

#时间戳
date +%s													#查看当下时间戳
date -d '2020-1-1' +%s						#查看指定日期的时间戳 默认显示00:00:00时的时间戳
echo '1577894400-1577808000'|bc		#1月2号时间戳减去1月1号时间戳，测算出1天共有多少秒，结果86400
date -d '2020-1-1 00:00:01' +%s		#查看指定时间点的时间戳
date -d @1577808000								#把指定时间戳转化日期
date -d @1577808000 +"%Y-%m-%d"		#把时间戳转为指定格式的日期
```

## 第二章 进程管理

##### ps命令

```bash
ps(process status)命令可以查看当前进程状态，STAT字段：D正在io不可中断 R正在运行 S休眠 T暂停或被追踪 X死掉 Z僵尸 <指优先级高 N指优先级低 L指有页被锁进内存 s是父进程 l是以线程运行 |是多进程 +是在前台运行

#名词解释
USER：启动程序的用户
PID：进程号
PPID：父进程的进程号
%CPU：占用cpu百分比
%MEM：占用内存百分比
VSZ：进程占用多少虚拟内存，单位KB
RSS：进程占用多少物理内存，单位KB，包含共享内存
TTY：运行的终端 ?指内核运行的终端  tty是机器运行的终端 pts是虚拟终端也就是远程连接的
STAT：进程状态
START：进程被触发者开启的时间
TIME：进程使用cpu的时间
COMMAND：命令的名称和参数，有[]表示内核态的进程，没有则是用户态进程

#生产实用
ps aux|head -1;ps aux|sort -k6nr|head				#查占内存前10的进程
ps aux|head -1;ps aux|sort -k3nr|head				#查看内cpu前10的进程
pgrep php -l																#过滤php进程,配合wc -l看子进程数量
ps auxf|grep [p]hp													#查看php相关父子进程
lsof -i :80																	#查看哪个进程在用80端口
lsof -p 2345																#查看pid为2345的进程打开了哪些文件

#进程管理命令
ps aux|head -1;ps auxf|grep [p]hp						#a所有与终端相关 x所有与终端无关 u以用户为中心 f层级关系
ps -ef|head -1;ps -ef|grep php							#-e所有进程 -f显示完整格式
ps axo user,pid,ppid,%mem,vsz,rss,command		#自定义显示项
ps auxf |grep [p]hp													#查看该进程的父子进程
ps axo user,command|grep mysql							#查看mysql进程详细的运行参数
ps -f -C php-fpm														#查看php-fpm命令所有进程，包含父进程,-C是group by command
ps -f -u apache															#查看apache用户起的所有进程，不含父进程，因为父进程是用root起的
yum install -y psmisc												#安装pstree命令的包
pstree -p 23015															#查看指定pid的子进程
pstree -p																		#查看所有进程的pid
pidof nginx																	#查看nginx进程的pid，一般最后一个是ppid
pgrep php -l																#查看php相关父子进程，第一行是父进程，-l显示进程
lsof -i :80																	#查看哪个进程在用80端口
lsof -p 2345																#查看pid为2345的进程打开了哪些文件
```

##### top命令

```bash
top命令可以动态查看进程状态,实际占用内存=RES-SHR

#名词解释
us：用户态，用户程序占用cpu比例
sy：系统态，cpu驱动硬件占用cpu比例，系统上下文切换的消耗，一般是进程太多导致
ni：友好值，通过调nice值来调整优先级
id：空闲cpu百分比
wa：等待输入输出的cpu百分比，wa高说明io很频繁
hi：硬中断，由硬件如磁盘网卡等产生，硬中断处理能迅速完成的事，属于上半部
si：软中断，由程序调用发生，软中断处理硬中断未完成的工作，是一种推后执行的机制，属于下半部
st：虚拟机占用物理机cpu占比
PR：优先级
VIRT：虚拟内存总量，VIRT=SWAP+RES
RES：常驻内存，单位kb，RES=CODE+DATA
SHR：共享内存，进程虽只用了几个共享库的函数，但包含整个库的大小，swap out后SHR会下降
S：进程状态

#top命令及交互命令
top -d3 -n5 -uroot  -b>1.txt			#-d刷新时间3秒 -n刷新5次退出 -uroot只看root -p10只看pid为10的
top -c														#显示完整的指令
O 进入排序页 - 输入需要排序字段对应的字母 - 按R倒序
M	按内存排序
P	按cpu排序
T	按cpu用时排序
N	按pid排序
n10只看前10 =取消只看前10
z	高亮显示
1 数据1查看所有cpu情况
i	只看状态为R的进程
s	修改刷新时间默认3秒
q	退出
f	选择显示项目 d确认显示 右方向键确认可移动 方向键来移动
unginx	查nginx用户的进程
k输入pid 再输信号来杀进程
r输入进程号 再输nice值，以此调整优先级 
```

##### pidstat命令

```bash
该命令用于监控指定进程占用系统资源情况，如cpu、内存、IO、任务切换、线程等，首次运行pidstat显示自系统启动开始到当下的各项统计信息，之后运行显示的是自上次运行该命令后的统计信息

yum install -y sysstat			#安装命令
pidstat -u 									#默认选项，统计cpu信息，%guest是虚拟机占多cpu百分比
pidstat -T TASK|CHILD|ALL		#-T指定监测内容,默认TASK监测单个进程的报告,CHILD是父子进程综合报告,ALL是两个都监测
pidstat -d									#统计io信息，kB_rd/s是该进程每秒读磁盘多少kb,kB_wr/s是写,kB_ccwr/s是截断的脏页
pidstat -r									#统计内存信息，VSZ是虚拟内存 RSS是非交换区里的物理内存，单位kb
pidstat -w									#统计任务切换情况，cswch/s是每秒自动上下文切换数 nvcswch/s是非自愿上下文切换数
pidstat -h	-l							#显示所有活动的任务,-l是显示命令及所有参数
pidstat -t -p 23015					#-t显示子进程信息，-p查看指定pid的统计信息 TGID代表主线程 TID是子线程
pidstat 2 10								#默认统计cpu信息，2是指定采样频率为每2秒1次 10是共统计10次后结束命令
```

##### 平均负载

```bash
平均负载：指单位时间内，进程状态为D(正在io不可中断)和R(正在运行)的平均进程数。理想状态是每个cpu的活跃进程数不大于3则性能良好，如果4核cpu，平均负载超3*4=12则略高；再如负载为9.03, 0.11, 0.09，cpu数为4，则9.34/4=2.35，性能良好。sy,si,hi,si任何一个超过5%都有问题

#排查高负载
进程数量过多时会有大量进程在等待cpu调度、I/O高时产生大量不可中断进程、cpu占用高等问题都会导致负载高
如果：%usr很高，%iowait很低，则是有某个进程大量占用cpu导致高负载
如果：%iowait很高，且%usr也高，则是进程占大量io导致负载高

1.先计算合理的负载值=cpu核心数*3
lscpu													#查看cpu几核
uptime												#查看平均负载

2.再查看cpu使用率、io等值
top -c												#使用率=1-%id，再按P找占用cpu最高的进程id
top -Hp 1418									#通过进程id找到占用cpu最高的线程
mpstat -P ALL 5								#判断负载高的原因，-P是cpu ALL是所有项 5是5秒刷新一次

3.找到根源的命令和进程
yum install -y sysstat				#安装命令
pidstat -d -l									#找到是哪个进程导致io高
pidstat -l										#找到是哪个进程导致cpu高

#压测工具：stress
stress -c 1 -t 600					#测cpu消耗用户态，反复计算随机数的平方根 -t 600指600秒后停止
stress -i 1 -t 600					#测io消耗内核态，反复调用sync()函数把内存写到磁盘
stress -m 1 -t 600					#-m 1是--vm 1，产生1个进程，反复malloc调用内存和free()来释放内存
stress -m 10 --vm-bytes 1G --vm-hang 100	#压测内存，10个进程，各分配1G，100秒后释放
watch -d uptime							#实时监测uptime命令输出结果，-d是高亮显示不同之处
```

##### 软硬中断

```bash
#硬中断
中断是系统用来影响硬件设备请求的一种机制，它会打断进程的正常调度和执行，然后调用内核中的中断处理程序来影响设备的请求

#软中断
事实上，为了解决中断处理程序执行过长的和丢失中断的问题，Linux将中断处理过程分成了两个阶段：
第一阶段：用来快速处理（硬中断）中断，它在中断禁止模式下运行，主要处理跟硬件紧密相关工作
第二阶段：用来延迟处理第一阶段未完成的工作，通常以内核线程的方式运行。
```

##### 进程信号

```bash
当程序运行为进程时，可以通过kill命令对该进程发送信号来管理进程

kill -l								#查看支持的信号，1是SIGHUP重启 9是SIGKILL强制中断 15是SIGTERM正常中止某进程
kill -9 30327					#发送信息 -9是强制中断进程 30327是pid 一般杀ppid
kill -SIGNUP $(ps aux|grep [r]syslogd|awk '{print $2}')		#重启syslog，重新读取配置文件
kill -15 30580				#正常关闭pid为30580的进程，不可以通过指定命令来关闭
killall -I php-fpm		#杀掉所有php-fpm命令的进程
pkill php							#模糊查杀所有关于php的进程
```

##### 进程优先级

```bash
系统是通过PR（priority）值来确定进程的优先级，PR值越低越优先执行。但是PR值是系统动态调整的，用户无权直接调整；但用户可以透过NI（nice）值来影响（只能是影响，最终由系统动态地决定）PR值，两者关系是：PR(new)=20+NI，固NI越小越优先
root用户可调整-20到19，普通用户0到19以避免普通用户抢占系统资源

#查看并修改nice值
top -c 								#用top命令查看并调整进程优先级
r输入进程号 再输nice值，系统自动计算新PR值=20+输入的nice值
ps axo user,pid,priority,nice,command|grep php	#命令查看pr值
renice -n -1 31520															#直接修改

#案例
Linux假死现象，比如ip能ping通，但ssh上不去，部署的nginx也打不开页面

当出现一个死循环的父进程，不断fork子进程，把文件描述符占满了，系统内存溢出后系统自动杀死子进程，父进程再fork新的子进程，循环往复，别的进程无法使用，可通过提升其它进程优先级，高于死循环的父进程
```

##### 后台进程

```bash
curl -I localhost:80 -H "Host: invoice.swatch.cn/" -H "X-Forwarded-Proto: https"

screen -S scp											#新建视窗并命名为scp
screen -d scp											#-d是detached使离线
```

## 第三章 内存管理

```bash
常用命令：free -m 、 top、slabtop、vmstat、pidstat
Mem
buff
cache
VIRT
RES
SHR
```

##### 磁盘性能

```bash
软硬中断，查进程

yum install -y sysstat
iostat -k	
```

## 第四章 磁盘管理

##### inode与mbr分区

```bash
#磁盘
磁道：盘片上的圆环，从外向内分别为0磁道、1磁道、2磁道
扇区：磁道上的最小单位是扇区，默认512字节(512byte=512B=0.5KB)
柱面：不同盘片的同一磁道构成的圆柱面

#文件系统
当磁盘被分区格式化后，会分为inode和block两部分，inode用于存放元数据和block指针，block用于存放真实数据，每个block只能存储一个文件，如果文件很大，就需要用于很多的block，系统通过该文件inode中的block指针找到磁盘上对应的block来读取数据，相当于block的索引。当新建一个文件时，系统会分配一个唯一的inode号，当用rm删除文件时也只是删除inode，只有磁盘再次写入时才会覆盖原来的block。
元数据:即属性信息，包括文件类型、权限、链接数、属主、属组、大小、atime、mtime、ctime等
block:操作系统读取硬盘时不是一个个扇区地读，而是一次性读取一个块(block)，块一般是4KB，即：4096字节/512字节=8个扇区

#调整block大小
注意：由于每个block最多可存放1个文件，当需要存放大量小文件时，inode号占用很多，但每个block的4KB空间都占不满，此时可以把block大小调小，为2的位数即可，如1K、2K。具体操作如下：

tune2fs -l /dev/vda1 |grep Block			#查看block大小,默认4096byte
fdisk -l |grep Sector									#查看扇区大小,默认512byte
mkfs.xfs -b 2K /dev/vg_nfs/lv_omega		#在逻辑卷格式化时指定block大小，也可指定为

#MBR分区表
位置: 0磁头0磁道1扇区
大小: 512字节=前446字节（mbr主引导记录）+中64字节（分区表）+后2字节（分区结束标识55AA）
分区: 适用于磁盘小于2TB的磁盘，分区类型MBR，主分区*4或主分区*3+扩展分区（逻辑分区+…），分区后需要保存后生效
设备名: sd字母后从a开始表示第一块磁盘，类推；数字从1开始表示该磁盘第1个分区，其它类推
注意：如果有3个主分区，则没有sdb4，因为第4个分区是整个拓展分区，拓展分区再分区是从sdb5开始

#单位
b：位（bit）比特 二进制存储 0或1，网络宽带用b为单位
B：字节（byte）1B=8b
utf-8字符集：1个汉字占3字节、1个英语字母占1字节、1个中文标点占3字节、一个英文标点占1字节
GBK字符集：1个汉字占2字节、1个英语字母占1字节、1个中文标点占2字节、一个英文标点占1字节
```

##### 磁盘分区挂载

```shell
lsblk												#查看块设备，/dev/sdb代表第2块磁盘；/dev/sdc2代表第3块磁盘的第2个分区
df -hT											#查看磁盘使用情况
fdisk /dev/sdb							#给磁盘分区
n新建分区，选择分区类型(p是主分区l是逻辑分区)，选择起始扇区(结束扇区用'+20G'表示)，p打印分区表，d删除分区，w保存退出
fdisk -l										#查看分区信息
mkfs.xfs /dev/sdb1					#把分区格式化为xfs文件系统
mount /dev/sdb1 /mnt				#挂载分区
mount -a										#不重启设备生效
umount -l /dev/sdb1 or /mnt	#卸载设备或目录都行 -l是lazy强制卸载
vim /etc/fstab							#永久挂载
挂载的设备 挂载点 文件系统类型 挂载参数 是否备份 是否检查				#查uuid用命令：blkid|grep vdc1
/dev/sdb1 /data  	xfs 		 defaults  0      0
是否备份指挂载后是否被dump备份，0是不备份，1是每天备份，2是不定期备份
是否检查指是否用fack检查硬盘，0是不检查，1是检查（挂载点为/时，必须填1），2是指完成检查后进程二级检查

brw-rw---- 1 root disk      8,  80 Jun 12 13:11 sdf
brw-rw---- 1 root disk      8,  96 Jun 12 13:05 sdg
```

##### lvm管理

```bash
LVM是Logical Volume Manager （逻辑卷管理）的简写。需要安装yum install -y lvm2
首先需要通过pvcreate命令把磁盘初始化，形成物理卷（physical volume）;
再把多个物理卷组成卷组(volume group)
然后从卷组里划分逻辑卷(logical volume),逻辑卷是最小单元为基本单元(physical extend)，默认4MB
最后格式化了才可以挂载使用

#常用命令
添加磁盘,添加好后可用lsblk查看到,添加磁盘方法如下：
在阿里云添加磁盘：ECS实例-云盘-创建云盘-挂载到实例
在Vmware添加磁盘：srv-swatch-nfs1右击选属性-硬件-添加硬盘(指定大小)-确定-然后去实例添加磁盘
执行以下命令可以不用重启添加磁盘
echo "- - -" >  /sys/class/scsi_host/host0/scan
echo "- - -" >  /sys/class/scsi_host/host1/scan
echo "- - -" >  /sys/class/scsi_host/host2/scan
echo "- - -" >  /sys/class/scsi_host/host3/scan
pvcreate /dev/vdb{1..3}										#把分区转分物理卷
pvcreate /dev/vdb													#也可以不分区，直接把整块磁盘转分物理卷
vgcreate vg_nfs /dev/vdb									#创建卷组,创建时必须最少指定一个物理卷
lvcreate -L 5G -n lv_omega vg_nfs					#从vg_nfs卷组中划分逻辑卷,-L指定逻辑卷大小，-n指定逻辑卷名称
mkfs.xfs /dev/vg_nfs/lv_omega							#格式化逻辑卷
mount /dev/VG_nfs/LV_swatch /nfs_swatch/	#挂载使用

当逻辑卷不够用时，需要扩容
vgs																							#通过vgs查看卷组内剩余空间
lvextend -L +1G /dev/mapper/vg_nfs-lv_omega -r	#-r表示扩容同时更新文件系统，若没加-r则lv_omega依然只能用5G
xfs_growfs /dev/vg_nfs/lv_omega									#忘加-r时，手动更新lv_omega的文件系统

当逻辑卷不够用，卷组内也没有多余空间时
pvcreate /dev/vdc																#先把新磁盘转成物理卷
vgextend vg_nfs /dev/vdc												#再把vdc加入vg_nfs卷组
lvextend -L +1G /dev/mapper/vg_nfs-lv_omega -r	#最后给逻辑卷扩容即可

当数据量下降，卷组空间剩余很多，需要减一块磁盘时
lsblk																						#通过lsblk查看该磁盘是否被逻辑卷占用，若没有则操作如下
vgreduce vg_nfs /dev/vdc												#从卷组中踢除vdc物理卷
pvremove /dev/vdc																#删除vdc物理卷后可拿掉磁盘或不用删物理卷，将其加入别的卷组

当需要去掉磁盘vdc，而vdc被逻辑卷lv_tissot占用，且正挂载使用
lsblk																						#查得lv_tissot同时占用vdb和vdc，且正挂载使用
pvmove /dev/vdc /dev/vdb &>pvmove.log &					#把vdc上的数据移到vdb, &>用于记录正常和错误日志，&放后台
vgreduce vg_nfs /dev/vdc												#移完数据后就没有逻辑卷占用vdc了，直接从卷组移除
pvremove /dev/vdc																#最后删除物理卷即可

当需要给卷组缩容
xfs(centos7默认文件系统)不支持缩容，ext4(centos6默认文件系统)缩容风险大，不提倡
umount /dev/mapper/domuvg-patrick
e2fsck -f /dev/mapper/domuvg-patrick
resize2fs /dev/mapper/domuvg-patrick 3G
lvreduce -L 3G /dev/mapper/domuvg-patrick
mount /dev/mapper/domuvg-patrick /home/patrickwu/data/

#物理卷管理
pvcreate /dev/vdc													#把磁盘或分区转为物理卷
pvmove /dev/vdc /dev/vdd									#把vdc的数据迁移到vdd，得vdc和vdd在同一个卷组且不可使用逻辑卷别名
pvremove /dev/vdc													#删除物理卷,在这之前需要先把对应的逻辑卷从卷组中踢除
pvs																				#查看所有物理卷
pvscan	--cache														#当磁盘重新分区后,UUID变化了但未写入lvm时报错，--cache来同步
pvdisplay																	#查看物理卷详情(pv名、UUID、大小、归属哪个VG、PE大小及使用量等)

#卷组管理
vgcreate vg_nfs /dev/vd{b,c}							#创建卷组
vgremove vg_nfs														#卷组删除，需要先卸载在用的逻辑卷
vgextend vg_nfs /dev/vdb2									#把物理卷/dev/vdb2加进卷组VG_nfs
vgreduce vg_nfs /dev/vdb3									#把物理卷/dev/vdb3从卷组VG_nfs中踢出
vgs																				#查看卷组

#逻辑卷管理
lvcreate -L 5G -n lv_omega_image vg_nfs		#从vg_nfs中划分逻辑卷，-L指定大小 -n指定名称
lvextend -L +2G /dev/vg_nfs/lv_omega_image -r 						#给逻辑卷增加2G -r表示扩容的同时格式化文件系统
lvextend -l +10%FREE /dev/mapper/vg_nfs-lv_omega_image -r	#把卷组内剩余空间的10%扩容进逻辑卷,无-r扩容不显
xfs_growfs  /dev/vg_nfs/lv_omega_image										#如果扩容时忘记加-r，就需要手动更新xfs文件系统
umount /omega_image																				#移除逻辑卷前需要先卸载
lvremove /dev/mapper/vg_nfs-lv_omega_image								#移除逻辑卷
```

##### swap分区

```bash
swap又称虚拟内存、交换分区等，当内存不够用时把不经常运行的进程踢进swap里，当需要运行该进程时再从swap里加载进内存，swap可临时充当内存使用，避免OOM随机杀进程，一般用未挂载的设备或空文件制作swap分区

查看
free -mh																							#查看目录使用内存和swap情况
swapon -s																							#查看swap在用的设备（文件还是磁盘，大小名称类型等）
配置
dd if=/dev/zero of=/swap.file bs=2M count=1024				#先准备2G的空文件(该文件不用提前创建),bs是blocksize
mkswap /swap.file																			#把该空文件制作为swap分区
swapon /swap.file																			#把该空文件加入到swap
swapon /swap.file1																		#需要扩展swap分区时，可再添加一个空文件进swap
恢复
swapoff -a																						#禁用所有swap
swapon -a																							#恢复swap到磁盘刚分区时的状态

补充知识：
物理内存剩余多少开始使用swap，是由内核参数中的vm.swappiness控制，参数范围0~100，数值越大表示越积极使用swap。
centos7上默认0,表示内存低于vm.min_free_kbytes = 45056(44M)时开始用swap，若设为60则表示剩余60%内存就开始用swap

查看
sysctl -a|grep swappiness															#查看当前生效的swappiness值
cat /proc/sys/vm/swappiness 													#同上
修改
sysctl -w vm.swappiness=0															#临时生效，重启失效。-w是write，表示修改
echo 60>/proc/sys/vm/swappiness												#效果同上，临时生效
sed -i '/vm.swappiness/cvm.swappiness=50' /etc/sysctl.conf	#想永久生效需要修改配置文件，/*/c是原地替换
sysctl -p																										#不重启加载配置文件，当前生效的参数也会被改
```

## 第五章 服务管理

##### 服务启停

```bash
systemctl list-unit-files |grep enabled		#查看系统开机自启的软件，centos6,7,8都可用
systemctl list-unit-files |grep enabled		#查看系统开机自启的软件，centos6,7,8都可用
#CenOS6
service tomcat start					#启动服务
service tomcat stop						#关闭服务
chkconfig tomcat on						#开机自启
chkconfig tomcat off					#取消开机自启
/etc/init.d/tomcat status			#查看运行状态
chkconfig --list							#查看运行级别
ifdown eth0										#停用eth0网卡，要去console控制台执行，否则远程连接的会夯住
ifup eth0											#启用eth0网卡
service network restart				#重启网卡

#CentOS7
/usr/lib/systemd/system 			#类似C6系统的启动脚本目录/etc/init.d/
/etc/systemd/system/ 					#类似C6系统的/etc/rc.d/rcN.d/
/etc/systemd/system/multi-user.target.wants/

#centos6改主机名
hostname 												#查看当前生效的主机名
hostname newname								#临时修改，重启失效
vim /etc/sysconfig/network			#永久修改，重启生效，配合临时修改的，则不需要重启就永久生效
第二行： HOSTNAME=newname
vim /etc/hosts									#修改本地域名映射,未知如果不操作这一步有什么影响，
在127.0.0.1 后加 newname

备注:
/etc/hosts用于配置ip与主机名的对应关系，局域网内ip唯一，配置好hosts后，可用主机名通讯，配置方法如下：
ip 主机名或域名 主机别名(可省略)  示例： 127.0.0.1 localhost.localdomain localhost
```

##### supervisor

```bash
supervisor工具可用于守护某个进程，当发现守护的进程挂了，自动拉起来

#安装配置
yum install -y supervisor
systemctl start supervisord
systemctl enable supervisord

cat >/etc/supervisord.d/nginx.ini<<EOF
[program:nginx]																	#指定进程名，supervisor通过这个名字来该进程
user=root
command=/usr/sbin/nginx -g 'daemon off';				#nginx默认在后端运行，-g拉到前端
priority=1000																		#优先级
autostart=true																	#supervisor启动时启动nginx
startretries=3																	#
autorestart=true
EOF
supervisorctl update														#重载配置文件

#基础命令
supervisorctl status [nginx]										#查看状态
supervisorctl reload														#重载服务
supervisorctl start nginx
supervisorctl restart nginx
supervisorctl stop nginx

#debian配置npc
cat>/etc/supervisord.d/npc.ini<<EOF
[program:npc]
user=root
command=./opt/npc/npc -server=www.zlwm115.com:8024 -vkey=r0u57xngxsgidthv -type=tcp
priority=1000
autostart=true
autorestart=true
startretries=3
EOF
```

##### 守护node进程

```bash
先安装nodejs
npm install forever -g
forever  start server/server.js 
forever list
```

## 第六章 目录和文件

##### 常用命令

```shell
#常用命令
du -sh *|sort -hr					#从大到小排列目录和文件
ll -thr										#从旧到新排列目录和文件,可用于找当前目录下的日志文件

#其它命令
ls -[thirdF1]	/etc				#time,human,inode,reverse,directory,long,oneline
ll -thr 									#按修改时间倒序显示
ll |grep ^d|wc -l					#查看目录总个数
ll -R|grep ^-|wc -l 			#递归查看所有文件数量
du -shx *									#查看此目录内所有文件和目录大小
du -h .|sort -hr|head			#查看前10个最大的目录
tree -[adF] -L 3	/etc		#all文件和目录,仅目录,F相当于ls -F,-L 深度为3层

1.操纵目录或文件
mkdir [-mvp]  目录名 			#mode -m 666 a 创建a目录时指定666权限,verbose显示过程,parents
cp -[ait] /tmp/a /opt/b		#-a等价于-rpd(递归,保留权限,复制软链接本身),-i 若要覆盖时提醒,-target 调换位置
mv -[fit] dir1 /tmp/dir2	#force,覆盖提醒，调换源目位置
ln -s 源 目标							#软链接
ln /a/a.txt .							#硬链接，让文件多个编辑入口

2.删除大量文件
rm -fr /opt/*										#文件数超10万就删不了，报错是参数过多
rsync -a --delete /tmp/empty .	#速度最快 -a 表示递归传输并保持文件属性 --delete源目一致，empty是源
find . -type f |xargs rm -f			#直接rm删不了，用find可删

3.rm删除时排除某些文件
shopt extglob						 	#查看
shopt -s extglob					#开启
shopt -u extglob					#关闭
rm -fr !(a.txt|b.txt)			#使用案例
rm -fr !(a*)
```

##### find查找

```bash
find /dev -type b|c|d|f|s|l|p   #b块设备、c字符设备、d目录、f普通文件、s套接字、l链接、p管道文件
find /opt -name "*.conf"				#查找opt下以.conf结尾的文件，-iname忽略大小写
find /opt -size +5M (-1k)	  		#+表示大于，-小于，无符号是等于，M必须大写,k要小写
find /opt -mtime +7							#+表示7天前，-表示7天内包括今天，无符号表示往前第7天
find /opt -mmin	 +120						#120分钟之前的，-mmin是内容修改，-amin是访问,-cmin是inode(元数据)的变化
find /opt -amin  +30            #查30分钟前被存取过的文件
find / -user tss -a -group tss	#-a表示且 -o表示或
find /etc -nouser -o -nogroup		#查找没有属主或没有属组的文件
find /etc -perm 644 -ls					#精确匹配权限为644的文件
find /etc -perm -644 -ls				#属主至少有6权限且属组至少有4且其它用户至少有4
find /etc -perm /202 -ls				#属主至少有2或其它用户至少有2权限
find /etc -perm -4000 -ls				#找有suid权限的文件
find /etc -perm -2000 -ls				#找有sgid权限的文件
find /etc -perm -1000 -ls				#找有sticky权限文件
find /etc -maxdepth 3 -type f		#查找3层目录
find /etc -links   +2           #查硬连接数大于2的文件或目录
find /etc -empty                #查找大小为0的文件或空目录

find . -name '*.txt' -ls						#长格式显示找到的文件，不可用-ll
find . -name "*.txt" -delete				#找到删除，只能删除空目录
find . -type f -exec rm -f {} \;		#找到删除， -exec command {} \;是固定格式
find . -type f -ok rm -f {} \;			#-ok作用跟-exec一样，只是每删1个文件都要问一次是否删除
find . -type f -exec cp {} /tmp \; 	#拷贝到/tmp目录下
find . -type f |xargs cp -t /tmp		#-t是调换源目文件位置
find . -name "2*"|xargs rm -fr			#删除文件和目录，找到的内容里不可有.和.. 有则跳过.会删别的
find . -inum 1179689 -delete				#通过inode删除文件,可删空目录,删不了非空目录,用于删以特殊字符命令的文件
find . -mtime +10 -ls|awk 'BEGIN{sum=0}{sum+=$2}END{print sum/1024/1024"G"}'  #计算查找出来的文件总大小
find /var/log/ -type f -size +2M -exec ls -lh  {} \;	#查找出来的文件以人类可读方式显示大小
find  /  -type f -ls |sort -k2nr|head -3							#查找本机里最大的3个文件
find /WRMS-PROD-MEDIA/cards -mtime +60 -type f |xargs tar czf /WRMS-PROD-MEDIA-ARCHIVE/`date "+%F"`.tgz && find /WRMS-PROD-MEDIA/cards/ -mtime +60 -delete				#find出来后压缩打包再删除
```

##### locate查找

```bash
locate用于根据文件名来搜索文件，原理是直接从/var/lib/mlocate/mlocate.db数据库查询，执行效率比find高得多，唯一的弊端是该数据库里所有文件的索引由定时任务每天凌晨3点多执行一次，所以当天新建的文件无法用locate找出来

yum install -y mlocate						#安装命令
updatedb													#更新索引后，可用locate找出更新索引前所有的文件
locate passwd											#查找所以命令包含passwd的文件
```

##### 查看大文件

```bash
当需要查看大文件时，先看文件大小查行数，只有less不用加载文件的全部内容，且搜索高亮

#常用命令
wc -l messages-20200308				#查行数（行数后面会显示文件名）
wc -l <message-20200308				#查行数（仅显示多少行，而且速度更快）
less -miN messages-20200308
/error  搜索  n下一个 N上一个

#命令详解
wc -l messages-20200308			#统计行数,可同时统计多个文件的行数、单词数、字符数
less -miN messages-20200308	#N 显示行号，i 搜索时忽略大小写，m 显示读取的百分比
less -f /dev/sda						#强制打开特殊文件，如目录、块设备文件、二进制文件等
按键：
/error (向下搜索error)  n(继续向下匹配)  N(向上匹配)  空格(下一页)  回车(下一行) b(上翻半页) f(下翻半页)
ma (当前位置标记为a)  'a(回到标记a的位置) 

more +100 -5 messages.log		#启动时加载整个文件，从第100行开始读，每屏显示5行
more +/data messages.log		#从文件中找第一个出现data字符串的行，并从该外前2行开始显示输出	
按键：=(输出当前屏首行行号) 空格 (下一页) b (上一页) q (退出)

cat 1.txt 2.txt>3.txt				#合并文件
cat -n message.log					#--number 显示行号，空行也编号，一次性读完，大文件千万不能用
cat -b message.log					#--number-nonblank 显示行号，空行不编号
cat -A message.log					#等价于-vET(ends tabs)，给显示出的内容结尾加$,tab用^I表示
tac message.log							#倒序显示
head -20 messages-20200308	#显示前20行，不指定则默认显10行
head -c 100 messages.log		#显示前100个字符
tail -30 messages-20200308	#显示最后30行
tail -c 100 messages.log		#显示后100个字符
tailf /var/log/nginx.log		#根据文件描述追踪，文件被删或改名就退出
tail -F /var/log/nginx.log	#根据文件名追踪,当文件被删或改名会有提醒，保持探测，等到相同文件名出现继续追踪
tailf /var/log/nginx.log		#等价于tail -f，不同的是tailf追踪时，如果文件不增长就不访问磁盘，省电省IO
ctrl +s	暂停追踪 ctrl+q 继续追踪 ctrl+c 退出
```

##### 字符三剑客

```shell
#正则表达式	
\			转义符，将特殊字符进行转义，忽略其特殊意义
^			匹配行首，^是匹配字符串的开始
$			匹配行尾，$是匹配字符串的结尾
^$		表示空行
.			匹配换行符之外的任意单个字符，包括空格、tab
*			匹配其前面的字符0次或者多次
.*		表示所有
[ ]		匹配包含在[字符]之中的任意一个字符
[^]		匹配[^]之外的任意一个字符
^[ ]	匹配以[]内任一字符开头
[a-z]	匹配[]中指定范围内的任意一个字符
?			匹配其前面的字符1次或者0次							 （扩展正则）
+			匹配其前面的字符1次或者n次							 （扩展正则）
|			或者																  （扩展正则）
()		匹配表达式，创建一个用于匹配的字符串			  (扩展正则)
{n}		匹配之前的项n次，n是可以为0的正整数			    (扩展正则)
{n,}	之前的项至少需要匹配n次								   (扩展正则)
{n,m}	指定之前的项至少匹配n次，最多匹配m次，n<=m	（扩展正则）

#特定字符				含义
[[:upper:]]		所有大写字母
[[:lower:]]		所有小写字母
[[:alpha:]]		所有字母
[[:digit:]]		任意一个数字			例：echo '123'|sed -n 's#[[:digit:]]#X#gp'  会把123改成XXX
[[:alnum:]]		所有的字母和数字
[[:space:]]		换行或tab等
[[:blank:]]		空格与tab
[[:punct:]]		所有标点符号   ! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ] ^ _ ` { | } ~.

#grep
grep -w 'root' /etc/passwd										#过滤完整包含root的行,root前后有空格符号等隔开.有字母过滤不出
grep -oi 'root' /etc/passwd										#-o是仅过滤出root，不包括该行的其它内容。-i忽略大小写
grep -o . /etc/passwd													#.代表任一字符，配合-o将文件passwd内所有字符显示为一列
grep -ri '*' /etc/cron*												#递归过滤
grep -c 'root' /etc/passwd										#统计包含root的行数
grep -c '^$' /etc/services										#统计空行数量
grep -n '^[0-9 a-Z]' /etc/selinux/config 			#过滤出以数字0-9或字母a-Z开头的行,-n标行号
grep -Ev '^$|^#' /etc/selinux/config					#过滤以$或#开头的行。-E支持扩展正则，-v取反，|表示或者
egrep 'abc|123' /etc/foo.txt									#过滤出包含abc或123的行，egrep等价于grep -E,|要用扩展正则
grep -Ev '^[$#]' /etc/selinux/config					#过滤以$或#开头的行
grep -Eo '[0-9]{16,17}[0-9X]{1}' id.txt				#-o仅过滤出匹配到的内容，{16,17}表示最少匹配16次，最多17次
grep -A2 '^r' /etc/passwd											#过滤以r开头的行及其后面2行，-A after -B before -C center
```

##### sed

```bash
#sed
sed [参数] '[范围] 操作' foo.txt
参数： -i修改文件 -n取消默认输出 -r拓展正则 -e执行多个操作 
范围： 1,4是1到4行 5,$是第5到末行 1;4是第1和第4行  2,+3是第2行加后3行  1~2是奇数行  2~2是偶数行 /sys/,+3是含sys及后3行 /\^a/,/b/
sed中有3个匹配字符 ^表示行首   $表示行尾  &表示匹配用s#zhou#&xy#g到的字符zhou

sed -n '/^Net/,/Updated/p' abc.txt 						#过滤出以Net开头到包含Updated的行
sed -nr '/xx|yy/p' abc.txt										#过滤包含xx或yy的行，并打印出来,-n取消默认输出 -r支持扩展正则
sed '1,5anewline' abc.txt											#从第1到5行，每行后面追加一行，内容为newline
sed '4,10d;20,$iaaa' passwd										#删除第4-10行，在20到末行的每行前插入一行，内容为aaa。$是尾行
sed '/etc/,+1d' passwd												#删除包含etc的行以及后面一行 +1表后面一行
sed -n '/^abc/,9p' passwd											#打印出从以abc开头的行到最后一行
sed -r '/X|#/s|D|S|g' sed.txt									#过滤出包含X和#的行，行内的D替换为S,g是替换所有,无g只替换首个D
sed -r '/X|#/s|D|S|3' sed.txt									#过滤包含X和#的行，g表示全局替换，3表示只把第3个匹配到的D换成S，不指定默认是1
sed -i '/root/s#$# end#'	passwd							#过滤出包含root的行，并在各行尾添加" end"字符，识别空格
sed -i 's#good#very &#' foo.txt 							#在单词good前添加very,&表示匹配到的字符，即本示例中的good
sed -i 's#zhou#& xiaoyong#g' foo.txt					#在单词zhou后面添加xiaoyong, &表示匹配到的任意字符
sed -i 's/^/#/g' foo.txt											#注释：给所有行的行首添加#符号，可用于注释某些行
sed -i 's/^even/#&/g' foo.txt									#注释：给even开头的行添加注释符号
sed -i '/useEslint/s#^#//#g' a.txt						#注释：给包含useEslint的行添加注释，注释符号为//
sed 's@^# \(Net\)@\1@'  c.txt									#去注释：去掉包含Net的行中的注释符号
sed -i 's/^# Net/Net/' c.txt									#同上，去掉行首的#
sed -i '/port/s|#||1' c.txt										#去掉包含port的行中第1个#号 1代表用前2个@@匹配到的第1个内容
sed -i '/^a/cNULL' sed.txt										#过滤出以a开头的行,并把该行原地替换为NULL,-i是修改内容非仅打印
sed -i.bak '/^a/s#A#B#g' sed.txt							#过滤出以a开头的行,并把行内的A替换为B,-i.bak是修改前做个备份
sed -ie '/^$/d' -ie '/^#/d' c.txt							#删掉空行和以#开头的行，-e表执行多次操作，跟i不能调换位置
sed -ie '/^$/d;/^#/d' c.txt										#-e配置;来执行多次操作
sed -i.bak '/^SELINUX=/cSELINUX=disable' /etc/selinux/config
sed -i 's#IPADDR#ip#w ip.txt' test						#把sed操作的文件内容保存到另外一个文件中，w表示保存，ip.txt文件名  
sed  -i '/Ethernet/r myfile' test							#在匹配Ethernet的行，读进来另一个文件的内容并插入到匹配Ethernet的行后  
sed = a.txt																		#在a文件中每一行的上一行显示本行行号 =是显示行号

#sed后向引用
原文是		url: jdbc:postgresql://192.168.50.121:5432/editor?currentSchema=editor
sed -nr '/url/s#(.*)//.*/(.*)#\1//172.16.0.1:5544/\2#gp' a.txt 			#修改ip和端口
ifconfig eth0|sed -nr '2s#.*t(.*)  n.*#\1#gp'												#过滤ip，前两个#中的第1个(.*)在后两个#中用\1表示

sed -i '$a\\n#By:zhouxy		At:2021-1-18 \nCmnd_Alias DEVCMD=/usr/bin/cp\nCmnd_Alias DEVNOCMD =/usr/bin/rm' /etc/sudoers
sed -i '1i/var/log/pgpool-II/pgpool.log\n/var/log/messages\n/var/log/secure' syslog
```

##### awk

```bash
#awk
格式：awk 'BEGIN{command} pattern{commands}END{commands}' 
行处理前用于打印表头、初始化变量、指定分隔符；主处理块逐行处理pattern匹配到的行，$0表示整行，$1表第一个field，以此类推，$NF表示最后一列。END是行处理后，用于生成报告等
内置变量：FILENAME文件名，
NR当前行号，FNR也是表当前行号，区别是当awk处理多个文件时，NR标第2个文件时，接第1个文件尾继续编号，FNR则重新编号。
RS 记录分隔符，即每行的分隔符，默认\n，也可用 -vRS=xx 来指定
FS 字段分隔符，即每字段的分隔符，默认空白字条，也可用 -vFS=xx来指定，简写为 -F xx
NF 字段数量，可用$NF表最后一列
ORS 输出行记录分隔符，默认\n，-vORS=xx来指定，
OFS 输出字段分隔符，默认空白字符 用 -vOFS=xx来指定，用于对结尾进行格式化处理 

行处理前
awk 'BEGIN{printf "名字\t课程\t得分\n"}{print$0}' marks.txt	 #打印表头
awk 'BEGIN{var="name";print var}'													 #定义变量
awk 'BEGIN{i=1;while(i<10){print i;i++}}'									 #变量初始化
awk 'BEGIN{FS=":";OFS="->"}{print$1,$NF}' /etc/passwd 		 #以:为分隔符，打印所有行的第1列和最后一列，输出																														分隔符改为->.冒号是普通字符，要用双引号
按行匹配
awk -F '[ /]' '{print $NF}' sed.txt						#以空格和/为分隔符，打印所有行的最后1列
awk 'NR==2;NR==6'  /etc/passwd								#取出第2行和第6行
awk 'NR>=1&&NR<4;NR==7,NR==$NR'  /etc/passwd  #取出第1到3行，以及7到末行
awk '/^Z/' awk.txt 														#匹配出Z开头的行，区分大小写
awk '$0 !~/^root/' /etc/passwd								#匹配开头不是root的行
按列匹配
awk '$3~/^3/' awk.txt													#过滤出所有行里第3列中以3开头的行
awk '$3~/[15]$/' awk.txt											#过滤出所有行里第3列中以1或5结尾的行
awk -F: '/^sshd/{print $3+10.99}' /etc/passwd	#运算表达式
awk -F: '$1~/root/ && $3<=15' /etc/passwd			#用户为root且uid小于15的行
awk -F: '$3>=1000{print $1}' /etc/passwd			#过滤出第三列大于1000的列，打印出第一个字段
awk -F: '$NF=="/bin/bash"' /etc/passwd				#过滤最后一列为/bin/bash的行
awk  -vOFS=','  '{print $1,$2}' reg.txt				#取出第1和第2列，并以逗号分隔的形式输出
条件判断
awk '{if( expr ){cmd1;cmd2} else if( expr ) {cmd }else{ cmd }}' 	#条件语句格式
awk '{if($3>300){print$2}else{print$1}}' a.txt										#条件表达式	
awk -F: '{if($3<1000&&$3>=0)i++}END{print i}' /etc/passwd					#计算系统用户数量
awk -F: '{if($3>=0&&$3<1000){xt++}else if($3>=1000){pt++}}END{print NR,xt,pt}' /etc/passwd
while循环
awk '{while(condition){cmd}}'																			#基本格式
awk 'BEGIN{i=1;while(i<=10){print i;i++}}'												#打印出1-10
for循环
awk -F: '{for(i=1;i<=10;i++){print$1}}' passwd										#每行打印10遍
awk 'BEGIN{for(i=1;i<=9;i++){for(j=1;j<=i;j++)printf " %d*%d=%-2d",j,i,j*i;print}}'		#打印乘法口诀
awk数组
seq 100|awk '{i+=$0}END{print i}'																	#i+=$0等价于i=i+$0，累计赋值,结果5050
awk -F: '{shell[$NF]++;}END{for (i in shell){print i,shell[i]}}' /etc/passwd|sort -k2nr|column -t
ss -ant|sed 1d|awk '{status[$1]++}END{for(i in status){print i,status[i]}}'
awk '{ips[$1]++}END{for(i in ips){print i,ips[i]}}' access.log |sort -k2nr |head |column -t
awk '{status[$10]++}END{for(i in status)print i,status[i]}' access.log |sort -k2nr|grep '^[0-9]'
awk '{path[$7]++}END{for(i in path)print i,path[i]}' access.log |sort -k2nr|head|column -t
awk '{i+=$11}END{print i/(1024^2)"MB"}' access.log
awk '{ips[$1]+=$11}END{for(i in ips)print i,(ips[i])/1024"KB"}' access.log|sort -k2nr|head
awk '{ips[$1]++}END{for (i in ips)if(ips[i]>3)print i,ips[i]}' access.log |sort -k2nr|column -t
awk -F '["]' '{print "echo " $0  " >> /tmp/logs/$(date -d @"$8" +\"%Y-%m-%d\").log"}' json.txt|bash
```

##### 排序去重截取

```bash
sort原理是把文件每行作为一个单位来比较，从首字符向后按ASCII码比较，升序输出
uniq是把相邻的两行去重，一般要先排序再去重，如果排序时指定了
tr是把标准输入一个字符一个字符得翻译，不可直接翻译文件，可用<或|

#常用命令
sort -k2nr -k4h gs.txt |column -t
sort -C  f.txt;echo $?
sort  a.txt|uniq -c	

排序
sort -k2nr -k4h gs.txt |column -t	#默认空格为分隔符，可用-t:指定:为分隔符，-k2表第2列,n按可读的数值大小,r倒序
google  110  5000  4G
sohu    100  4500  30kb
baidu   100  5000  300kb
guge    50   3000  2G
sort -C  f.txt;echo $?						#检查是否排好序，是则返回0，否返回1
sort -n num.txt										#sort默认按字符排序,遇到数字时要加-n，让按数字大小排序，否则11会比2小
du -sh * /etc|sort -hr						#数字带单位时，要用-h指定按实际大小排序，否则10kb会比1G大。-r是倒序
sort -u a.txt -o sort.txt					#-u去重，识别-k指定的列，排序后去掉后面重复行；-o把排序的结果另存到sort.txt
-f 忽略大小写排序 -M按月份排序 -b忽略前面所有空行 -d表示对本域按照字典顺序排序（即，只考虑空白和字母）
去重
sort  a.txt|uniq -c								#排序后统计各个重复行数量，uniq只能统计相邻的行，所以要先排序
uniq -d a.txt											#只显示重复的行 --repeated
uniq -u b.txt											#只显示不重复的行 --unique

截取字符
cut -c5-8 tc.txt									#截取每行的第5到第8个字符
cut -c-5 tc.txt										#截取第1-5个字符，最前或最后可用-表示
cut -d: -f2,3 f.txt								#-d指定冒号为分隔符，-f截第2和3个段；默认以tab分隔
cut -d ' ' -f2,3 sort.gs					#指定空格为分隔符

tr(translate)只对标准输入翻译，不能翻译文件
cat a.txt|tr -d  "\n"							#去掉换行符
cat a.txt |tr " " :								#空格换为冒号
cat a.txt |tr a-z A-Z >da.txt			#小写转大写，另存为da.txt
```

##### 重定向

```bash
执行任何命令都会打开3个文件: /dev/stdin  /dev/stdout /dev/stderr
0表示标准输入:	<
1表示标准输出:	1> 或> 
2表示错误输出:	2>

wc -l <host_ip.sh 							#计算行数，不显示文件名
:>message.log										#清空文件
ip a >ip.txt										#命令执行结果重定向到ip.txt
cat a.txt b.txt>c.txt						#按顺序把a和b文件合并
(ls;date)>a.txt									#把多条命令输出的结果重定向到a.txt
du -sh /* 1>right.out 2>err.out	#正确输出定向到right.out 错误输出定向到err.out
du -sh /* 2&>a.txt							#把1和2都定向到a.txt
du -sh /* &>/dev/null						#把1和2都定向到空
du -sh /* >/dev/null 2>&1				#功能同上，仅写法不同
dd</dev/zero>a.file bs=1M count=20	#等同 dd if=/dev/zero of=/tmp/a.txt bs=1M count=20
cat >>a.txt<<"EOF"							#当内容里有变量又不希望被解析时，EOF需要加双引号或单引号
`data`
EOF

#tee
echo hello |tee a.txt b.txt 		#把屏幕输出复制一份到a.txt和b.txt

```

## 第七章 用户及权限

##### 用户和组管理

```bash
/etc/passwd			#用户名:密码占位符:UID:GID:注释:家目录:登陆shell
/etc/shadow			#用户名:密码:变更时间:最少使用天数:最长使用天数:到期前提醒:到期后提醒:失效时间:保留字段
/etc/group			#组名:组密码:组GID:附加成员
/etc/gshadow		#组名:组密码:组管理员:附加成员
/etc/login.defs	#新建用户的默认配置文件，包含umask，系统和普通用户uid的起始编号，
/etc/default/useradd	#跟login.defs一起决定新建用户的默认参数，包含家目录初始环境变量、指定家目录位置、登陆shell
/etc/motd				#登录系统提示的界面
#用户管理
id zhouxy																							#查看用户uid gid和组；不接用户名表示查当前用户
id zhouxy -gn																					#查看用户zhouxy的基本组名 -gn是groupname
useradd dba -u88 -g88 -Ga,b  -c "nb" -d /home/dd			#-g指定基本组，-G附加组 -c注释 -d家目录
useradd www -u777 -g777 -M	-s /bin/bash							#-M不建家目录 -s指定登陆shell
useradd www -r																				#-r建系统用户，无家目录，有登陆shell
userdel zhouxy																				#删除用户，保留家目录
userdel zhouxy -r 																		#删除用户以及家目录和邮件
usermod www -u666 -g666 -c"newB"  -s /sbin/nologin		#-u改uid -g改gid -c改注释 -s改登陆shell
usermod zhouxy -md /bbb																#-m配合-d把原家目录内容迁到新目录/bbb(自动新建的)
usermod zhouxy -g dba																	#修改用户zhouxy的基本组为dba
usermod zhouxy -G wheel																#修改用户zhouxy的附加组为root组，覆盖之前的附加组
usermod	zhouxy -aG dba,ops														#-a配合G追加附加组，无-a则删掉之前的组
usermod zhouxy -l jonny																#把zhouxy的用户名修改为jonny
usermod -L zhouxy																			#锁定用户
usermod -U zhouxy																			#解锁用户
cp /etc/skel/.bash* /home/zhouxy											#当环境变量默认文件被删，可从skel中复制，以恢复bash
#密码
echo "123"|passwd --stdin zhouxy											#非交互式给用户设密码
passwd zhouxy  回车再输新密码													 #交互式给用户设密码
passwd zhouxy -l																			#锁定zhouxy用户密码，在当前shell也无法改密码
pkill -kill -t pts/0																	#锁定后把用户踢下线
passwd zhouxy -u																			#解锁
echo $(echo $RANDOM|md5sum |cut -c 5-14) |tee pass.txt| passwd --stdin dba	#生成10位的随时密码
chage -l jonny.zhou																		#查用户最新改密码时间,密码过期和失效时间,账户过期时间
chage -m 2 jonny																			#改完密码后2天内不能再改
chage -M 60 jonny																			#改完密码后，此密码60天后过期,账户依然保留
chage -E '2020-9-1' jonny															#指定账户有效期到9月1号，9月1号后系统再无此用户了
chage -d 0 jonny																			#强制用户下次登陆要改密码
vim /etc/pam.d/system-auth														#修改设置密码时要求的长度，默认8个字符
password  requisite   pam_cracklib.so try_first_pass retry=3 minlen=3
#组管理
groupmod ops -g1024 -ndba															#ops是现有的组名，-g指定新gid -n指定新组名
groupdel dba																					#基本组内有用户时无法删基本组，附加组内有用户也可删组
groups zhouxy																					#查看用户所属的组，也可用id zhouxy
grep dba /etc/group																		#查看dba组有哪些成员
gpasswd dba -a zhouxy																	#把zhouxy加入dba组
gpasswd dba -d zhouxy																	#把zhouxy从dba组删除
gpasswd -M user1,user2,user3 dev											#把用户批量加入dev组
#切换用户
su - zhouxy																						#切换用户zhouxy,加载环境变量
su zhouxy																							#切换用户不加载环境变量
su - zhouxy -c 'head -2 /etc/passwd'									#以zhouxy的身份执行命令
#用户授权
sudo -i																								#直接切换到root用户，用zhouxy用户身份每次都要sudo
sudo -u tomcat sh /opt/aa.sh													#以tomcat用户身份执行sudo命令，tomcat是系统用户
sudo -l																								#查看当前用户的sudo权限
visudo																								#编辑sudo配置文件，等价于vim /etc/sudoers
使用者账号	登陆者来源主机名=(可切换的角色) 可执行的命令			 #可执行的命令一定要是绝对路径，否则语法错误
zhouxy       ALL=(ALL)   NOPASSWD:ALL									#zhouxy使用sudo执行命令时不用验证zhouxy的登陆密码
%dba				 ALL=(ALL)	 !/usr/bin/passwd root				#用%表示dba是个组，!表示禁用的命令
gpasswd dba -a zhouxy 或者用 usermod zhouxy -aG dba		#把用户添加进dba组就可以继承dba组的sudo权限
gpasswd dba -M user1,user2,user3											#清除dba组成员，一次性添加user1,2,3
使用别名来配置权限组
User_Alias DBA=zhou,wu,wang														#User_Alias固定写法不能变
Cmnd_Alias CMD=!/usr/bin/passwd,!/usr/bin/rm					#命令可用正则
DBA	ALL=(root)	CMD																		#调用配置的别名
visudo -c																							#检查sudo配置文件有无语法错误
tail -f /var/log/secure 															#sudo日志审计

[root@srv-traning ~]# write zhouxy pts/2							#发私信
hello:	you're excellent! ^C
wall "I'will shutdown web1"														#发广播
```

##### 用户分配权限示例

```bash
#先建组再批量建用户并赋密码且要求下次登陆必须验证当前密码再改密码
groupadd dev
for i in `cat name.list`;do useradd $i -G dev;echo 'test'|passwd --stdin $i;chage -d 0 $i;chage -E '2021-1-19' $i;done
ueradd zhouxy -G dev && echo 'test'|passwd --stdin zhouxy

#分配sudo权限
sed -i '$a\\n#By:zhouxy		At:2021-1-18 \nCmnd_Alias DEVCMD=/usr/bin/cp\nCmnd_Alias DEVNOCMD =/usr/bin/rm' /etc/sudoers
sed -i '$a%dev    ALL=(ALL)       NOPASSWD:DEVCMD,!DEVNOCMD' /etc/sudoers

#赋予执行kubectl权限
for i in `cat name.list`;do mkdir /home/$i/.kube;cp ~/.kube/config /home/$i/.kube;chown -R $i.$i /home/$i/.kube;done
```

##### 文件普通权限

```bash
文件的权限是针对文件内容，r可以读取内容，w可以对文件内容进行增删改，x是指可以通过./a.txt被系统执行。
注意:w需要配合r，否则编辑后覆盖原内容；普通用户的x需要配合r才能执行，root仅有w也能执行

目录的权限是针对该目录下的文件或子目录，r可以读取目录结构和清单，w可以增删改移该目录下文件或目录，x是可以cd进该目录。
注意:r读取目录里内容需要配合x，否则只能看到文件名，看不到详细信息；w也需要配合x才能修改，只不过没有r看不到目录里的内容

#修改权限
chmod u=rwx,g=w,o=-	user.txt	#直接赋权法，-代表无任何权限。u是user g是group o是others a是all
chmod =- /opt/passwd					#等同a=-，a代表所有用户 -是无任何权限
chmod u+x /dir/user.txt				#加权法，给属主加执行权限。
chmod +x /opt/omega						#等同a+x，表示给所有用户加x权限
chmod a-x /dir/user.txt				#所有人去掉执行权限
chmod -R 0755 abc.sh					#数字赋权法，数字前的0可省略。4是r 2是w 1是x ，-R是递归赋权
chown -R www.www /code				#递归设置属主和属组 可用www.www或www:www来表示
chown -R www.	/code						#仅修改属主为www
mkdir -m 444 /opt/omega				#新建目录时指定权限，新建文件时不能这么干

#umask
用户新建文件或目录时默认的权限是通过umask调节的，文件的默认权限=文件的最大权限666-umask;目录的默认权限=目录最大权限777-umask；如果计算结果是奇数，为避免赋执行权限，把奇数加1作为默认权限。umask的值在/etc/profile文件中定义:uid大于199的用户其umask为002，其余用户的umask为022
umask									#查看当前用户umask
umask 044							#临时修改当前用户umask值
umask涉及文件可能还有：/etc/bashrc /etc/profile ~/.bashrc ~/.bash_profile
```

##### 文件特殊权限

```bash
#特殊权限
suid:给某个命令如/usr/bin/cat赋予了suid权限后，可以让others组的用户借用该命令属主(root)的权限，可看/etc/passwd
stid:给目录如/share赋予stid权限后，任何用户在/share下新建的文件或子目录的属组会继承/share的属组，属主仍是用户自己
sbit:给目录如/share设置了sbit后，任何用户(除root外)都只能删除、改名、移动自己建的目录或文件,sbit对二级目录无效

chmod u+s /usr/bin/cat				#给cat命令赋予suid权限，cat命令的权限为-rwsr-xr-x
chmod 4755 /usr/bin/cat				#效果同上，s位于属主的执行权限的位置，属主有执行权限用s表示，无执行权限用S表示
chmod g+s /share							#给目录设置sgid，s位于属组的执行权限位置，有执行权限显示为s，无则显示为S
chmod 2770 /share							#2000表示sgid
chmod o+t /share							#给目录设置sbit，用t表示sbit，位于其它组的执行权限位置，有x显示为t，无x显示为T
chmod 3770 /share							#给目录/share同时设置sgid和sbit，一般用于配置共享目录

示例：给dba组的用户新建共享目录，
groupadd dba									#先准备dba组
gpasswd dba -a zhao						#分别把zhao，qian，sun等用户加入dba组，此时dba是用户zhao、qian、sun的附加组
mkdir -m 3770 /share					#同时赋予sgid和sbit，继承/share目录的属组且可能删自己建的目录和文件
chown .dba /share							#给/share目录属组改为dba

#特殊属性
给文件或目录设置属性，其权限凌驾于wrx之上，常见属性有：
a只能追加不能删改覆盖 i锁定文件或目录不许增删改 s从block上删数据后用0填充(无法恢复) u删文件时保留block(可恢复数据)
S是sync只要有写入系统立马把修改的内容从内存写到磁盘 A指不更新Atime b不更新存取时间 c压缩后存放 D检查压缩文件中的错误

chattr +a /mine				#给目录增加a属性，仅作用于/mine目录下一级的文件和子目录，不可删改移，二级目录不受约束
chattr +a nginx.log		#给文件添加a属性，只能用>>追加内容，cat等查看内容，不能删文件 不能vim编辑 不能用>覆盖
chattr +i /mine				#锁定/mine目录和/mine下的目录及文件不允许增删改名移动，但文件里的内容可以增删改，不约束下级
lsattr /opt						#查看/opt目录下所有文件和目录的属性,不递归检查下一层目录
lsattr a.txt b.log		#查看指定文件的属性


setfacl -m u:zhouy:wrx /tmp/a.txt			#赋权
getfacl /tmp/a.txt										#查看
如果授权不成功，需要在挂载nfs时添加acl属性
```

##### 环境变量

```bash
/etc/profile				#但凡需要输入密码登陆进来的都是登陆式shell，登陆式shell都会先读取本配置用以设置全局环境变量如										判断PATH中要不要sbin、USER、HISTSIZE、umask等，另外还会调用/etc/profile.d/*.sh

/etc/profile.d/*.sh	#定义有bash界面颜色、语系、vi和ls及which等的别名，如果需要配置所有用户共享的别名可在此配置，其中的lang.sh调用/etc/locale.conf来定义语系，bash_completion.sh调用/usr/share/bas*

~/.bash_profile			#登陆式shell读完全局配置就会读用户个人的环境变量配置文件,该文件通过export PATH新增了用户家目										 录下的~/bin让各用户可以有自己的命令，判断有~/.bashrc就优先执行~/.bashrc

~/.bashrc						#此配置最终都会执行，可以把各自的偏好设置写在这里，该文件先定义用户设置的alias，再判断是否											有/etc/bashrc，有就调用。如果本文件被删，命令提示符坏了，就用cp /etc/skel/.bashrc  ~/

/etc/bashrc					#根据不用uid设置umask及PS1的值，还会调用/etc/profile.d/*.sh
										
~/.bash_logout			#退出当前shell时执行
~/.bash_history			#记录历史命令，用户登陆时读取本文件，并载入内存，用户就可以通过history命令查看历史命令

source /etc/profile.d/alias.sh	#重新加载环境变量,因为/etc/profile和.bash_profile都只在登陆时执行一次
. /etc/profile.d/alias.sh				#效果同上

#查看、定义变量
env															#查看所有的环境变量，如HOME SHELL HISTSIZE MAIL PATH LANG RANDOM等
set															#查看所有环境变量和自定义变量，如PS1不是环境变量，自定义变量不能被子进程继承
export													#查看环境变量
name=zhouxy											#定义一个自定义变量name
unset name											#取消对name变量的定义，可用于取消自定义变量和环境变量
export name											#把自定义变量转为环境变量，这样新开一个bash或在shell脚本里也能继承这个变量
declare -x name									#把name转为环境变量，效果同export，declare用于声明变量类型
declare +x name									#把name从环境变量转为自定义变量
declare -r name									#锁定定量为只读，不可修改也不可unset，重新登陆自动取消
declare -i sum=10+20						#指定sum类型为数值，echo $sum结果为30
declare -p sum									#列出sum的类型，后面不加sum则列出所有变量的类型

#语系变量
locale -a												#查看系统支持的语系
locale													#查看当前语系，当设置了LANG或LC_ALL则其他语系变量都会被这两个变量替换掉
echo $LANG											#查看主语言环境
echo $LC_ALL										#查看整体语系环境，如时间系统、币值、数字、时间系统等
LANG=en_US.UTF-8								#临时修改，在子进程里也生效，重新登陆失效
export LC_ALL=zh_CN.UTF-8				#临时生效，LC_ALL是需要转为环境变量才生效
export LC_ALL=en_US.UTF-8
```

##### 历史命令

```bash
#历史命令
history -c											#清除内存中所有历史记录
history -r											#从.bash_history读取历史记录到内存，当内存被清空时，可从该文件恢复命令记录
history -w											#把当前内存中的记录写入.bash_history，内存被清空则会覆盖性清空该文件
history -c;history -w						#清除内存里记录并写入文件来清空文件中的历史记录
history 10											#查看最后10条历史记录
history -d2											#删除第2条记录
export HISTFILE=~/.bash_history	#指定记录历史记录的文件
export HISTSIZE=10000						#指定通过history命令能显示多少条历史命令
export HISTFILESIZE=1000				#指定.bash_history文件能存多少条历史命令
export HISTCONTROL=ignorespace	#仅忽略空格开头的命令，ignoredups是仅忽略连接且相同的命令，ignoreboth指都忽略
export HISTTIMEFORMAT="%F %T" 	#定义历史命令显示格式为添加时间戳形式
export HISTIGNORE="rm:pwd:ls"		#忽略所有rm\pwd\ls命令，不记录到history里

#历史命令审计脚本
1.在/etc/rsyslog.conf里加入一行  user.*      /var/log/cmd_track.log
sed -i '$a\\n#By:zhouxy At:2021-1-17\nuser.* /var/log/cmd_track.log'  /etc/rsyslog.conf
systemctl restart rsyslog

2.写脚本
cat>/etc/profile.d/cmd_track.sh<<'EOF'
export HISTTIMEFORMAT="[`whoami`@`who am i 2>/dev/null|awk '{print $NF}'|sed -e 's#[()]##g'`] "
export PROMPT_COMMAND='\
if [ -z "$OLD_PWD" ];then
export OLD_PWD=$PWD;
fi;
if [ ! -z "$LAST_CMD" ] && [ "$(history 1)" != "$LAST_CMD" ]; then
logger  "$(history 1)";
fi ;
export LAST_CMD="$(history 1)";
export OLD_PWD=$PWD;'
EOF
source /etc/profile.d/cmd_track.sh

3.结果样式
Aug 23 22:34:59 web1 zhouxy:  211  [root@10.0.0.1 /root] ll

4.缺陷
不能记录sudo -i ，重复的或空格开始的命令只能二选一，没有命令提示符


#通过prompt实现
cat >/etc/profile.d/cmd_track.sh<<'EOF'
USER_IP=`who am i 2>/dev/null|awk '{print $NF}'|sed -e 's#[()]##g'`
PROMPT_COMMAND='date "+[ %Y%m%d-%H:%M:%S $LOGNAME from $USER_IP ]: `history 1 |cut -c 8-`" >>/var/log/cmd_track.log'
EOF
source /etc/profile


#示例
sed -i '$a\\n#By:zhouxy At:2021-1-17\nuser.* /var/log/cmd_track.log'  /etc/rsyslog.conf
systemctl restart rsyslog
cat>/etc/profile.d/cmd_track.sh<<'EOF'
export PROMPT_COMMAND='\
[ -z "$OLD_PWD" ] && export OLD_PWD=$PWD;
[ ! -z "$LAST_CMD" ] && [ "$(history 1)" != "$LAST_CMD" ] && logger  "$(history 1)";
export LAST_CMD="$(history 1)";
export OLD_PWD=$PWD;'
EOF
source /etc/profile.d/cmd_track.sh
chattr +a /var/log/cmd_track.log
```

## 第八章 包管理

##### 压缩包管理

```bash
#命令常用
gzip /etc/nginx/conf.d/123.com.conf							#原地压缩
gzip -d /etc/nginx/conf.d/123.com.conf.gz				#原地解压
zip -q /tmp/a.zip a.txt													#压缩文件
unzip -q /tmp/a.zip  -d /opt/										#解压目录到指定位置，有屏显
tar czf /tmp/etc.tgz /etc	 											#以相对路径打包，-P使用绝对路径打包
tar czf /tmp/etc.tgz --exclude=/etc/nginx /etc	#打包时排除nginx目录
tar xf  /tmp/etc.tgz  -C /opt/etc								#指定解压路径

#zip压缩文件或目录（不常用）
zip -q /tmp/ab.zip a.txt  b.txt							#压缩多个文件到指定目录，会有屏显，-q去屏显 --quiet
zip -rq /opt/abc.zip abc/										#打包abc目录到/opt下，-r是递归，无-r则只打包空目录
zip -T /tmp/abc.zip													#检查压缩包是否完整 -T  --test
unzip -l /tmp/abc.zip												#查看压缩包列表 --list
unzip abc.zip  -d /opt/											#解压目录到指定位置，有屏显
zcat file.zip 															#查看zip压缩文件的内容，不能看压缩目录		

#gzip只能压缩文件，原地压缩删源文件，不可指定压缩路径，无交互
gzip /etc/nginx/conf.d/www.123.com.conf			#把123.com.conf变成123.com.conf.gz
gzip -d /etc/nginx/conf.d/www.123.com.conf  #变回来
gzip -r /etc/nginx/conf.d/									#逐个压缩conf.d下所有文件， -r 递归
gzip -dr /etc/nginx/conf.d/									#conf.d下文件全部解压
zcat /etc/nginx/conf.d/default.conf.gz			#查看压缩文件内容

#tar打包
tar czPf /tmp/etc.tgz /etc									#打包会删根，-P使用绝对路径打包
tar tf /tmp/etc.tgz 												#查看包内容，不加-P的tar包里文件路径都不在/下
tar xf  /tmp/etc.tgz  -C /opt/etc						#指定解压路径
tar czf 99.tgz --exclude=1.txt --exclude=4.txt  {1..6}.txt		#可排除多个文件
tar czf 99.tgz --exclude-from=undo.list {1..6}.txt						# --exclude-from=文件  排除文件内列表
tar czfX 99.tgz undo.list  {1..6}.txt													#X 指定排除文件里的列表

语法：tar [-zjxcvfP] filename 
c   创建新的归档文件
x   自动选择解压模式,对归档文件解压缩
t   列出归档文件里的文件列表
v   输出命令的归档或解包的过程
f   指定包文件名，多参数f写最后
C   指定解压目录位置
z   使用gzip压缩归档后的文件(.tar.gz)
j   使用bzip2压缩归档后的文件(.tar.bz2)
J   使用xz压缩归档后的文件(tar.xz)
X   排除多个文件(写入需要排除的文件名称)
h   打包软链
P   连带绝对路径打包
czf 打包tar.gz格式
cjf 打包tar.bz格式
cJf 打包tar.xz格式
--hard-dereference  //打包硬链接

关于使用绝对路径打包 tar Pczf
当使用绝对路径打包，解压时不用-C指定目标位置时，才会覆盖原目录
当使用绝对路径打包，解压时使用-C指定目标位置时，系统自动去掉包内文件路径前的根号
当使用相对路径打包，不用-C就压缩到当前目录，用-C就直接解压到指定目录
```

##### 软件包管理

```bash
#常用命令
yum provides unzip								#连网查看命令属于哪个包，前提是yum仓库得有这个包
yum list zabbix*	 								#查看本机源所有能装的包，一般配合grep用
rpm -qa zabbix*			   						#查看系统中已安装的所有RPM软件包列表（模糊查询）
rpm -qa |grep zabbix							#如果zabbix*查不到就用grep
rpm -qf file1|commands   					#查询配置文件或命令属于哪个已安装的rpm软件包	
yum install zabbix					  		#安装zabbix	
yum localinstall *.rpm						#安装本地rpm包，再从其它源安装依赖
sed -i s/keepcache=0/keepcache=1/g  /etc/yum.conf										#开启yum缓存
yum clean packages																									#清理/var/cache/yum下的软件包
find /var/cache -iname "*.rpm" |xargs cp -t ~/zabbix-4.0.19/				#将缓存的rpm包拷贝到其它目录

#其它命令
yum makecache fast												#配置好yum源后，提前把软件包信息在本地缓存一份，以提高安装软件速度
yum clean all															#清理yum缓存，包括headers, packages等

yum repolist															#查看启用的仓库列表
yum repolist all													#查看启用的和没启用的仓库
yum-config-manager --enable php-webtatic	#启用yum仓库，此命令需要安装yum-utils包
yum-config-manager --disable php-webtatic	#禁用某仓库
yum list zabbix* 													#查看本机源所有能装的包，配置grep用
yum info zabbix40													#查看rpm包详细信息，包括未安装的包，rpm -qi只能查已安装的包

yum provides 	tree												#连网查看命令属于哪个包，前提是yum仓库得有这个包
yum install zabbix					  						#安装zabbix	
yum reinstall zabbix	  		   						#重新安装（软件有损坏时，正常安装会提示已安装，需要用重装）
yum localinstall *.rpm										#安装本地rpm包，再从其它源安装依赖
yum check-update													#检查可以更新的包，已安装的有颜色，未安装的无颜色
yum update -y tree 												#更新包，若没有指定包，则直接更新所有包，包括内核，非常危险
yum history																#查看历史命令
yum history info 3					 							#查看某历史命令详细信息
yum history undo 3					 							#撤销执行过某条历史命令，不能回滚
yum groups list														#查看已安装及未安装的软件组
yum groups install -y Security Tools			#安装整组软件包
yum groups remove -y Security Tools				#删除整组软件包
yum remove nginx													#删除rpm包，包括非系统必需的依赖一起删
yum erase	 nginx													#删除rpm包

rpm -ivh *.rpm														#安装rpm包，不下载依赖
rpm -Uvh *.rpm														#升级安装
rpm -qi zabbix-get			  						  	#查看指定软件的详细信息
rpm -qf /usr/bin/tree   	 								#查看某个命令属于哪个包，绝对路径，本地要有这个命令
rpm -qf /usr/share/doc/tree-1.6.0					#查询配置文件属于哪个包
rpm -ql zabbix-get   						  				#查询指定软件包所安装的目录、文件列表
rpm -qc zabbix-get				 	  						#查询指定软件包的配置文件
rpm -qd zabbix-get    										#查询指定软件包的帮助文档
rpm -q --scripts zabbix-web  							#查询rpm包安装前和安装后执行的脚本
rpm -e zabbix-agent												#删除包
```

##### Debian 软件包

```bash
#换清华源，华为源太慢
mv /etc/apt/sources.list /etc/apt/sources.list.bak
cat>/etc/apt/sources.list<<EOF
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-updates main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security buster/updates main contrib non-free
EOF
apt update
apt install apt-transport-https ca-certificates
```

## 第九章 远程连接

##### ssh远程登陆

```bash
#ssh常用命令
ssh-keygen -t rsa -P ''	-f ~/.ssh/id_rsa								#生成公私钥,-t指定算法 -P指定密码 -f指定路径
ssh-copy-id -i ~/.ssh/id_rsa.pub  root@139.196.185.235	#用命令发公钥,没有authorized_keys文件则自动创建
ssh root@139.196.185.235 -p 22  												#登陆到服务端，本地用户和远程用户相同时可省去用户名
ssh 139.196.185.235 'free -h'														#不登陆远程执行命令
ssh root@139.196.185.235 -v															#开启调度，追踪ssh连接过程
yes|pv|ssh root@139.196.185.235 'cat>/dev/null'					#实时ssh网络吞吐量测试，需先yum install -y pv
ssh srv-swatch-omega-wechat1														#如果想使用别名来登陆就需要在客户端上做如下配置
vim /etc/ssh/ssh_config.d/swatch.ssh_config							#修改配置
host srv-swatch-omega-wechat1
  hostname 192.168.20.95
  port 40022
  hostKeyAlias srv-swatch-omega-wechat1
/etc/init.d/sshd reload																	#重载服务
```

##### scp远程传输

```bash
#从ssh1上传到主机
scp -P 40022 image001.png ncadmin@srv-swatch-tissot-eco-web2:/home/ncadmin

#从主机下载到ssh1
scp -P 40022 tissot_web1_log.tgz  jonny.zhou@ssh1.service.chinanetcloud.com:/home/jonny.zhou/

#从mac上拉取ssh1的文件
scp -P 40022  jonny.zhou@ssh1.service.chinanetcloud.com:/home/jonny.zhou/get_users* /Users/cncuser/Desktop/
```

##### rsync远程同步

```bash
#rsync
rsync [OPTION]... SRC [SRC]... DEST
-a           					#归档模式传输, 等于-tropgDl
-v           					#详细模式输出, 打印速率, 文件数量等
-z           					#传输时进行压缩以提高效率
-r           					#递归传输目录及子目录，即目录下得所有目录都同样传输。
-t           					#保持文件时间信息
-o           					#保持文件属主信息
-p           					#保持文件权限
-g           					#保持文件属组信息
-l           					#保留软连接
-P           					#显示同步的过程及传输时的进度等信息
-D           					#保持设备文件信息
-L           					#保留软连接指向的目标文件
-e           					#使用的信道协议,指定替代rsh的shell程序
--exclude=PATTERN   	#指定排除不需要传输的文件模式
--exclude-from=file 	#文件名所在的目录文件
--bwlimit=100       	#限速传输
--partial           	#断点续传
--delete            	#让目标目录和源目录数据保持一致
--password-file=xxx 	#使用密码文件
```

##### clush

```bash
clustershell是轻量级的集群管理命令，只需要在server1上安装即可，

#安装
yum install -y clustershell
#配置（集群分组，集群内免密访问）
cat>>/etc/hosts<<EOF											#配置hosts
192.168.82.131 server1
192.168.82.132 server2
192.168.82.133 server3
EOF
ssh-keygen  															#配置免密
ssh-copy-id  server1
ssh-copy-id  server2
ssh-copy-id  server3
cat>/etc/clustershell/groups<<EOF					#配置clustrshell的组，手动创建groups文件
all:server[1,2,3,4]
kafka:server[1,2,3]
postgresql:[1,2]
EOF

#使用
clush -a 'ip a'									#-a指定集群内所有主机，相当于-g all
clush -g kafka 'ip a'						#-g指定执行命令的组
clush -w server1,server2 'ip a'	#-w指定某主机，多个主机逗号隔开
clush -a -b 'ip a'							#-a的结果每行前都有主机名，-b用来合并同一主机返回的结果
clush -a -x server1 'ip a'			#-x排除某主机，多主机逗号隔开
clush -a -X kafka 'ip a'				#-X排除某主机组，多个组逗号隔开
clush -g mqtt -c a.txt --dest /opt		#-c是--copy，拷贝文件到远程主机，--dest指定远程主机上的目录
clush -g mqtt -c dir1 --dest=/opt/dir2		#-c可用于递归拷贝目录，含隐藏文件。dir2不存在时，把dir1改名为dir2
clush -a --rcopy /foo.txt --dest=/root #--rcopy是拷贝远程主机的文件到本地，文件名.server1 文件名.server2
```

## 第十章 定时任务

##### 配置定时任务

```bash
修改配置，添加环境变量
vim /etc/crontab
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
:/root/bin

ll /var/spool/cron/						#查看有哪个用户用了定时任务
crontab -eu ncadmin						#编辑定时任务，等同/var/spool/cron/ncadmin
crontab -lu ncadmin						#查看ncadmin的定时任务
crontab -ru ncadmin						#删除ncadmin用户的定时任务
crontab -r										#删除root用户的所有定时任务
echo "zhang3">>/etc/cron.deny	#禁止用户zhang3使用定时任务
tailf /var/log/cron						#查看定时任务执行日志

3,15 8-11 */2 * * ls   				#每隔两天的上午8点到11点的第3和第15分钟执行
0 23-7/1 * * * ls     		 		#晚上11点到早上7点之间，每隔一个小时执行
*表任意时间段都执行
-表时间段
,表不连续的时间
*/表每隔多久执行一次
```

##### rsyslog

```bash
系统日志如内核、计划任务、邮件、错误信息、网络服务、用户登陆等服务的日志都是通过守护进程rsyslog服务产生，其配置文件是/etc/rsyslog.conf

#日志内容
事件发生日期与时间 产生该事件的主机名 启动该事件的服务名或命令、函数  实际内容

#配置rsyslog
*.info;mail.none;authpriv.none;cron.none  /var/log/messages	#除邮件、认证、定时任务外的信息写入message
authpriv.*                                /var/log/secure		#记录需要账号密码如login、ssh、su等信息
mail.*                                    -/var/log/maillog	#-是把日志先存进buffer，日志太多不便直接写硬盘
cron.*                                    /var/log/cron			#定时任务所有信息写入/var/log/cron
*.emerg                                   :omusrmsg:*				#:*表所有用户,所有服务最严重日志广播给所有用户
uucp,news.crit                            /var/log/spooler	#记录uucp和news的crit等级及更严重的级别的信息
local7.*                                  /var/log/boot.log	#.*表示所有级别的信息 .=info表示只要info信息
user.*                                    /var/log/cmd_crack.log #user是指用户可通过logger来记录日志

#信息等级
7 debug		#调式时产生的数据
6 info		#基本信息
5 notice	#比info更值得注意的正常信息
4 warn		#警示信息，可能有问题但不影响程序运行
3 err			#重大错误，程序已经不能运行
2 crit		#比error还要严重
1 alert		#比crit还要严重
0 emerg		#快要宕机了

#logger命令用法
logger -p user.info 
logger -p syslog.info "backup.sh is starting"			#-p指定服务和等级，syslog是rsyslogd程序产生的日志
logger -t zhouxy "it is userd by zhouxy"					#为信息打标签
```

##### logrotate日志切割

```bash
logrotate主要用于管理系统及各程序产生的日志文件，可以自动替换、压缩、删除、邮寄等操作。一般系统的日志文件直接在/etc/logrotate.conf里配置，第三方软件产生的日志在/etc/logrotate.d/下新建一个文件来配置

#配置logrotate
[root@jonny-training ~]# vim /etc/logrotate.d/nginx 
/var/log/nginx/*log /var/log/tomcat/*log{
		create 644 www root     #转存时建立新文件，权限644，属主www 属组root
		daily								    #每天转存一次，weekly表每周1次，monthly表每月1次
		copytruncate            #截断当前日志，新日志写入新文件，无此选项时需要重启服务，否则日志还是往旧文件里写
		rotate 5                #保留多少个日志，0为无备份,以数字为准
		missingok               #如果日志不存在，不报错
		notifempty              #如果日志文件为空，则不转储，默认是ifempty(空文件也转储)
		minsize 15M             #如果日志不超过15M，本次rotate不执行
		size 50M								#如果日志超过了1G,没到logrotate的时间也自动执行，默认是bytes,也可用K、M、G
		compress                #通过gzip压缩转储以后的日志,以*.gz结尾，默认是nocompress不压缩
		delaycompress           #延迟压缩,必须配合compress一起用，否则无效。表示本次切割的文件到下次切割时才压缩
		dateext                 #转存的文件以当天日期为后缀，不设置则用默认文件名：access_log.1 access_log.2
		dateformat -%Y%m%d%H.%s #使用了dateext才有效，用于修改默认的日期格式(-%Y%m%d).当日志每天切割多次时才用得上
		sharedscripts						#开启共享脚本，配合postrotate或prerotate用
    postrotate							#postrotate/endscript里包含的是转储以后需要执行的命令
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true	#
    endscript								
		prerotate/endscript			#放在prerotate/endscript中间表示在转储以前需要执行的命令，两个关键字必须单独成行
		errors 243074974@qq.com #转储时的错误信息发送到指定的Email地址
		mail zhou012@yeah.ne    #把转储的日志文件发送到指定的E-mail地
		nodelaycompress         #覆盖delaycompress选项，转储同时压缩
		nocopytruncate          #备份日志文件但是不截断
		nocreate                #不建立新的日志文件
		nomail                  #转储时不发送日志文件
		noolddir                #转储后的日志文件和当前日志文件放在同一个目录下
		olddir /tmp             #转储后的日志文件放入指定目录,必须和当前日志文件在同一个文件系统
		tabooext                #不转储指定扩展名的文件,缺省扩展名：cfsaved,.disabled,.dpkg-dist等
}

#配置完后测试
logrotate -d /etc/logrotate.d/nginx 		#-d是调度，默认自带-v属性，会显示过程，用于排错或了解程序执行过程
logrotate -f /etc/logrotate.d/nginx 		#-f是强制执行，这样手动执行后配置自动生效，之后按定时任务走
logrotage -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf #-s指定状态(哪些rotate执行到哪了)

#logrotate的定时任务及定时任务执行时间点
1.logrotate的自动执行切割是用定时任务完成的
cat /etc/cron.daily/logrotate
#!/bin/sh
/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf	#指定状态文件并执行
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"		#报错时记录日志
fi
exit 0

2.该定时任务执行时间是在/etc/anacrontab里配置的
cat /etc/anacrontab
START_HOURS_RANGE=3-22			# 执行时间为每天3:22分
RANDOM_DELAY=45							# 随机延时时间，最长45分钟，也就是最晚执行时间是4:05分

3.通过查看当前logrotate状态可查看执行rotate的历史记录
cat /var/lib/logrotate/logrotate.status

#如果想每12个小时执行rotate，可以重写脚本放定时任务里执行

```

## 第十一章 虚拟机

##### 新建虚拟机

```bash
#配置双网卡（外网：eth0，内网：eth1）
虚拟机与宿主机间的通讯是通过vmnet来实现，vmnet是虚拟网卡，其功能相当于交换机，使用vmnet1来实现仅主机模式网络连接，通过vmnet8来实现nat网络连接，可新建自定义的vmnet2对应eth0来实现外网访问，vmnet3对应eth1来实现内网访问，配置方法如下

1.修改网卡命名规范
光标移到Install CentOS7时按tab，最下方输入：net.ifnames=0 biosdevname=0

2.添加虚拟网卡
vmware fusion - 偏号设置 - 网络 - 点加号来添加vmnet2 - 勾选使用NAT，填写子网IP（10.0.0.0），这个子网ip是vmnet2的网段，再去掉默认勾选的DHCP - 应用，同理新建vmnet3，子网ip为192.168.0.0

3.查看默认网关
Mac: cat /Library/Preferences/VMware Fusion/vmnet2/net.conf
Windows: 网络共享中心 - vmnet2 - 查看详情

4.配置虚拟机网卡
虚拟机centos7.8 - 设置 - 网络适配器1（eth0）- 选择vmnet2
虚拟机centos7.8 - 设置 - 网络适配器2（eth1）- 选择vmnet3

5.进入操作系统配置网卡信息
vi /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=10.0.0.3
NETMASK=255.255.255.0
GATEWAY=10.0.0.2
DNS1=223.5.5.5
DNS2=223.6.6.6
vi /etc/sysconfig/network-scripts/ifcfg-eth1
TYPE=Ethernet
BOOTPROTO=static
NAME=eth1
DEVICE=eth1
ONBOOT=yes
IPADDR=192.168.0.3
NETMASK=255.255.255.0

#配置逻辑卷
注意/boot分区不可使用逻辑卷，否则报错，常用分区挂载如下
centos-root
centos-swap
```

##### 系统初始化

```bash
#克隆的虚拟机改ip和host脚本
cat>~/host_ip.sh<<'EOF'
#!/bin/bash
OLD_IP=$(awk -F= '/IPADDR/{print$2}' /etc/sysconfig/network-scripts/ifcfg-ens18 )
OLD_UUID=$(awk -F= '/UUID/{print$2}' /etc/sysconfig/network-scripts/ifcfg-ens18 )
NEW_UUID=$(uuidgen)
. /etc/init.d/functions
read -p 'Please enter your new IP: ' NEW_IP
read -p 'Please enter your new hostname: ' NEW_HOST
sed -i "/^IPADDR/s#${OLD_IP}#${NEW_IP}#g" /etc/sysconfig/network-scripts/ifcfg-ens18
sed -i "/^UUID/cUUID=${NEW_UUID}"         /etc/sysconfig/network-scripts/ifcfg-ens18
nmcli c reload ifcfg-ens18
if [ 0 -eq 0 ];then
   action "network" /bin/true
else
   action "network" /bin/false
fi
hostnamectl set-hostname ${NEW_HOST}
new_name=localhost.localdomain
bash
if [ == ];then
   action "hostname" /bin/true
else
   action "hostname" /bin/false
fi
EOF

#克隆完虚拟机修改UUID和HWADDR
nmcli con show																#查看UUID
nmcli d show|grep HWADDR											#查看hwaddr
uuidgen																				#生成新的UUID
vim /etc/sysconfig/network-scripts/ifcfg-eth0	#修改UUID和HWADDR
UUID=df7b0a59-83dc-42fc-8ed4-08c50b2dd381
HWADDR=
nmcli c reload ifcfg-eth0														#重启网卡，立即生效
```

##### 内核升级

```bash
#centos7升级内核到指定版本
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org										#导入密钥
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm			#安装源
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available							#查看可安装的内核
yum --enablerepo=elrepo-kernel install -y kernel-ml														#安装内核

#查看系统可用的内核
grub2-set-default 0																														#配置默认启动的内核为最新
grub2-mkconfig -o /boot/grub2/grub.cfg																				#将上一步配置写入配置文件
reboot																																				#重启生效
rpm -qa|grep kernel																														#查看系统安装了的内核
package-cleanup --oldkernels																									#只保留3个版本，清理其它

#centos8升级内核到指定版本
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org										#导入密钥
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm			#安装源
yum --enablerepo=elrepo-kernel install kernel-ml															#安装内核
grub2-set-default 0																									#设置以新的内核启动，0表示最新的内核
grub2-mkconfig -o /boot/grub2/grub.cfg															#将修改写入配置文件，重启生效
reboot
```

##### 内核模块

```bash
#加载内核模块
modprobe br_netfilter											#手动加载内核模块，立即生效，重启失效
cat>/etc/modules-load.d/cri.conf<<EOF			#自动加载模块的配置文件
overlay
br_netfilter
EOF
cat>/etc/modules-load.d/ipvs.conf<<EOF	
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
EOF
systemctl restart systemd-modules-load		#配置立即生效，不执行restart的话重启机器这些模块也会自动加载

注意：如果加载模块需要带参数，就需要修改/etc/modprobe.d/cri.conf文件，内容示例：
options nf_conntrack_ftp ports=555

#内核模块相关命令
lsmod														#查看系统所有模块名、模块大小及使用者，0代表无使用者
modprobe ip_vs									#加载模块
modinfo ip_vs										#查看模块详情
modprobe -r nf_conntrack_ftp		#删除模块
```

##### 内核参数

```bash
sysctl -a													#查看所有内核参数
sysctl -p /etc/sysctl.d/k8s.conf	#查看指定文件里配置的内核参数，-p不指定文件时默认读/etc/sysctl.conf
sysctl -w vm.swappiness=40				#临时调整内核参数，立即生效，重启失效
cat>/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system										#从所有内核配置文件中加载内核参数，包括如下目录，从上到下读取，不覆盖已读参数
              										/run/sysctl.d/*.conf
              										/etc/sysctl.d/*.conf
              										/usr/local/lib/sysctl.d/*.conf
              										/usr/lib/sysctl.d/*.conf
              										/lib/sysctl.d/*.conf
              										/etc/sysctl.conf
```





```bash
journalctl --since 15:00:00 -u kubelet


```

