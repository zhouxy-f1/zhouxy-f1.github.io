## 什么是高可用

高可用是启动2台完全相同的业务系统，一台故障，另一台自动接管。可用keepalived软件来实现，keepalived是基于VRRP协议（虚拟路由冗余协议）。起初为解决网关单点故障问题，在两台网关外新增一个虚拟的节点，配置VMAC与VIP，用户发送报文到VMAC，由VMAC转发到后面可用的网关，

## 核心概念

确定主备：投票选举 确定优先级

抢占式：A宕机切到B，A恢复了抢回master；

非抢占式：A宕机切到B，A恢复了就等着B宕机才能切回来

脑裂：B无法检测到A的状态，以为A宕机，就抢了master，这时出现两个主机绑定VMAC和VIP	

## 配置高可用

##### 解读配置文件

```shell
#全局配置
global_defs {
#标识身份
   router_id lb01
}
vrrp_instance VI_1 {
#标识角色状态
    state MASTER
#网卡绑定接口
    interface eth0
#本角色归属的组id    
    virtual_router_id 50
#优先级    
    priority 150
#监测间隔时间    
    advert_int 1
#认证    
    authentication {
#明文认证     
        auth_type PASS
#明文密码        
        auth_pass 1111
    }
    virtual_ipaddress {
#虚拟的VIP地址    
    10.0.0.3
    }
}
```

##### 配置主

```shell
[root@lb01 ~]# cat /etc/keepalived/keepalived.conf 
global_defs {
   router_id lb01
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
    10.0.0.3
    }
}
```

##### 配置备

```shell
[root@lb02 ~]# cat /etc/keepalived/keepalived.conf 
global_defs {
   router_id lb02
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
    10.0.0.3
    }
}
```

##### 抓包结果

![image-20191014152634806](https://tva1.sinaimg.cn/large/006y8mN6gy1g7xrefyep4j30q50ozdn5.jpg)



## 配置非抢占式

两个节点的state都要配置为BACKUP，且都要加上nopreempt，优先级一高一低

##### 配置主

```shell
...
vrrp_instance VI_1 {
	state BACKUP
	priority 150
	nopreempt
...
}
```

##### 配置备

```shell
...
vrrp_instance VI_1 {
	state BACKUP
	priority 100
	nopreempt
...
}
```

## keepalived在nginx上应用

1、Nginx默认监听在所有的IP地址上

2、用户将域名解析到VIP上就可以了



## 脑裂问题

##### 产生的原因

1、服务器网线松动

2、服务器硬件故障

3、主备都开启了firewalld



##### 影响

两台主机争抢VIP，VIP来回切换，此时网站仍可访问

##### 解决办法

随机关掉一台lb的keepalived服务

## nginx服务挂掉问题

如果nginx服务死掉，keepalived不会飘移走，VIP依然绑定在这个lb，此时就是宕机

解决办法：写监控脚本（nginx服务停止，自动启动，如果启动失败，马上停keepalived）再把监控脚本写入keepalived里面执行

##### 1.写脚本

```shell
#建目录
[root@lb01~ ]# mkdir /server/script -p
#写脚本
[root@lb01 /server/script]# vim check_web.sh
#!/bin/bash
nginxpid=$(ps -C nginx --no-header|wc -l)
if [ $nginxpid -eq 0 ]; then
        systemctl start nginx
        sleep 3
        nginxpid=$(ps -C nginx --no-header|wc -l)
        if [ $nginxpid -eq 0 ]; then
                systemctl stop keepalived
        fi
fi
#加执行权限
[root@lb01 ~]# chmod +x /server/script/check_web.sh 
```

##### 2.脚本放入keepalived中执行

抢占式时，只需要在master中执行脚本，非抢占式时backup中也需要执行此脚本，因为非抢占式不检测nginx的话，lb1恢复，lb2的nginx挂了，停掉lb2的keepalived不会自动到lb1

```shell
[root@lb01 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
        router_id lb01
}
#定义脚本，每5秒执行一次，要高于check_web.sh脚本执行的时长，否则.sh没执行完，这里执行就无效
vrrp_script check_web {
        script "/server/script/check_web.sh"
        interval 5
}
vrrp_instance VI_1 {
        state MASTER
        interface eth0
        virtual_router_id 50
        priority 150
        advert_int 1

authentication {
        auth_type PASS
        auth_pass 1111
    }
virtual_ipaddress {
        10.0.0.3
        }
#调用并执行此脚本
track_script {
        check_web
        }
}
```

























