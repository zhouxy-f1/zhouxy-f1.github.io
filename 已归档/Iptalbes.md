## 第一章 iptables

##### 简介

```bash
iptables是基于包过滤的防火墙工具，可以对流入和流出服务器的数据包做很精细的控制，主要工作在OSI的二、三、四层。防火墙是层层过滤的，按照规则的配置顺序，从上到下一行一行地匹配，匹配到就执行（匹配拒绝也是匹配到），匹配到了就不再向下匹配，匹配不到就找下一层，都匹配不上就执行默认数据包规则，默认规则是最后才执行的

注意：内网尽量不要开iptables，因为会消耗cpu性能，尤其在大并发下。如果想监控七层可以用squid代理结合iptables来做

#四表五链
filter表：默认表，用于过滤流入流出本主机的数据包，真正负责本机防火墙的功能，包含3个链，分别是
1.INPUT链：负责过滤进入主机的数据包
2.FORWARD链：负责转发流经本机的数据包，内核转发，用net.ipv4.ip_forward=0控制
3.OUTPUT链：负责处理从本机发出去的数据包，一般不配置
nat表：不流经本主机，只做转发，用于转换源目ip和端口，即转发数据包。nat表包含三个链，分别是
1.PREROUTING: 当数据包到达防火墙时，更改该数据包的目标ip和端口，相当于重写收件人地址
2.POSTROUTING:当数据包离开防火墙时，更改该数据包的源ip和端口，相当于改写回信地址，配置为本路由器公网ip就可以共享上网
3.OUTPUT:更改本主机发出数据包的目标地址
mangle表：主要负责修改数据包中特殊的路由标记，不怎么用得上
raw表：有限级最高，设置raw时一般是为了不再让iptables做数据包的链接跟踪处理，提高性能

注：iptables包含四张表，表里包含链chains，链里包含过滤规则policy


#四表五链工作流程
																	___ filter INPUT => filter OUTPUT__
流入 => nat PREROUTING => 路由决策=>|																	 |=> nat POSTROUTING =>数据流出
																	|__________filter FORWARD__________|
```

##### 启停iptables

```bash
iptables是启停是通过加载内核模块来实现的，加载完直接生效，但重启iptables或重启服务器失效。在CentOS7里直接用systemctl start firewalld来启动iptables

#查看状态
/etc/init.d/iptables status				#CentOS6里查看防火墙状态或用 service  iptables  status
systemctl status firewalld				#CentOS7里查看防火墙状态
iptables-save											#快捷查询状态，6和7通用。有任何输出则表示启动，无任何输出则表示未启动

#启动iptables
systemctl start firewalld					#CentOS7里开启iptables
/etc/init.d/iptables start				#CentOS6里开启iptables,如果启动失败，提示no config file
lsmod |egrep 'nat|filter|iptable'	#查看是否加载到内核，如果没有则执行如下加载内核命令，iptables自动启动
modprobe ip_tables
modprobe iptable_filter
modprobe iptable_nat
modprobe ip_conntrack
modprobe ip_conntrack_ftp
modprobe ip_nat_ftp
modprobe ipt_state
```

##### iptables命令

```bash
iptables -t filter -A|-I|-D INPUT -p tcp -s 10.0.0.1 -i eth0 -j ACCEPT|REJECT|DROP
-t: 指定操作哪个表，不指定则默认filter表
-A(-I或-D):指定增或删动作，-A是在指定链的规则末尾添加新规则，-I是在指定链的规则首行插入规则，-D是删除指定规则
INPUT: 指定操作的链，也可指定为OUTPUT、FORWARD等其它链
-j: --jump指定匹配后的动作 ACCEPT是匹配后无动作 REJECT是匹配后回应拒绝 DROP是匹配后丢弃
-i: 后接网卡名称 匹配从这块网卡流入的数据
-o: 后接网卡名称 匹配从这块网卡流出的数据
-p: 匹配协议,如tcp,udp,icmp
--dport num 匹配目标端口号
--sport num 匹配来源端口号


#查看规则
iptables -nL											#查看默认的filter表规则 -n是以数字显示  -L是list列表显示
iptables -nL --line-numbers				#查看规则时给规则加序列号，从1开始，以后删除规则时可用序列号来指定删哪条规则

#清除规则
iptables -F	INPUT									#-F是--flush，不指定表的话清除所有表的规则(除默认规则外)
iptables -X												#清除用户自定义的链（默认的五链外用户自定义的链）
iptables -Z												#对计数器清零
iptables -D INPUT	2								#删除INPUT表第2条规则，第几条是通过--line-numbers参数查出来的
iptables -D NAT 2									#删除NAT表第2条规则

#匹配规则
1.按来源ip匹配
iptables -I INPUT 2 [!] -s 10.0.20.0/24 -j DROP 								#2是在第二行插入 !是取反
iptables -I INPUT -p tcp ! -s 10.0.0.1 -j DROP									#除跳板机外，全部拒绝
2.按端口匹配
iptables -I INPUT -p tcp --dport 80 -j DROP											#匹配单个端口
iptables -I INPUT -p tcp --dport 80:85 -j DROP									#匹配多个连续端口
iptables -I INPUT -p tcp -m multiport --dport 21,22,23 -j DROP	#匹配多个不连续端口
iptables -A INPUT -p tcp --sport 22:80													#匹配来源端口号是22到80的包

3.按协议匹配
iptables -I INPUT	-p icmp --icmp-type any -s 10.0.20.95 -j DROP	#禁ping
iptables -I INPUT -p udp -j DROP																#禁udp协议

4.按网卡匹配
iptables -A INPUT -i eth0																				#-i是input,指定从哪个网卡进来的流量
iptables -A FORWARD -o eth0																			#-o是output,指定从哪个网卡出去的流量

5.配置默认规则
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP
```

##### 规则持久化

```bash
因为当前iptables规则都是写在内存里的，重启全部失效

#首次保存
iptables-save>/etc/sysconfig/iptables																		#简易操作方法,首次用> 之后用>>
/etc/init.d/iptables save																								#保存配置
cat /etc/sysconfig/iptables																							#默认的配置文件

#保存后怎么修改
vim /etc/sysconfig/iptables																							#直接修改配置文件
/etc/init.d/iptables reload																							#重载生效

#恢复配置文件里规则到内存
iptables-save>/tmp/iptables_2019.7.2																		#备份
iptables-restore </tmp/iptable_2019.7.2																	#恢复
```

##### 示例：配置企业防火墙

```bash
1.配置允许远程连接
iptables -F
iptables -X
iptables -Z
iptables -A INPUT -p tcp --dport 22 -s 10.0.20.0/24 -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

2.配置默认规则
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT
iptables -P FORWARD DROP

3.配置自己人能连接
iptables -A INPUT -s 10.0.20.0/24 -p all -j ACCEPT
...

4.对外提供服务
iptables -A INPUT --dport 80 -j ACCEPT																	#开放80端口
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT				#允许ftp的包通过

5.保存iptables配置
iptables-save>/etc/sysconfig/iptables																		#简易操作方法,首次用> 之后用>>
/etc/init.d/iptables save																								#保存配置
cat /etc/sysconfig/iptables																							#默认的配置文件

6.修改iptables
vim /etc/sysconfig/iptables
/etc/init.d/iptables reload
```

