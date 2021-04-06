## OpenVPN概述

##### 原理

```bash
虚拟专用通道，在internet开辟一条加密通道，来访问内网，既保障安全又节省成本

#用户连接vpn
1.先在vpn服务器生成证书和密钥，把公钥分发给用户（windows、mac、linux等）
2.用户拨vpn，先验证密钥，再验证用户名和密码
3.通过之后vpn服务器从地址池取一个IP：10.8.0.3给用户，用户通过这个IP访问vpn私网

#用户访问vpn私有网络
拨号连接成功后，连到vpn外网，经内核转发连到内网网卡

#用户访问后端集群
用户访问web时，在web上抓包发现数据包能进不能出，web无法把信息回传给10.8.0.3，那么在vpn服务器用firewall把10.8.0.3封装成172.16.1.10来访问web，原路返回包，就能正常通讯
```

![image-20200325225605579](https://tva1.sinaimg.cn/large/00831rSTgy1gd6keq7t4sj30ld08jwfk.jpg)

## 部署vpn服务端

```bash
#先安装生成密钥的工具，存放到/opt/easy-rsa目录下，目录下要生成vars文件来验证身份
yum install -y easy-rsa	
mkdir /opt/easy-rsa
cp /usr/share/doc/easy-rsa-3.0.6/vars.example /opt/easy-rsa/
cat>/opt/easy-rsa/vars<<EOF
if [ -z "$EASYRSA_CALLER" ]; then
        echo "You appear to be sourcing an Easy-RSA 'vars' file." >&2
        echo "This is no longer necessary and is disallowed. See the section called" >&2
        echo "'How to use this file' near the top comments for more details." >&2
        return 1
fi
set_var EASYRSA_REQ_COUNTRY     "cn_only"
set_var EASYRSA_REQ_PROVINCE    "shanghai"
set_var EASYRSA_REQ_CITY        "pudong"
set_var EASYRSA_REQ_ORG         "tking"
set_var EASYRSA_REQ_EMAIL       "zhou012@yeah.net"
set_var EASYRSA_NS_SUPPORT      "yes"
EOF

#初始化生成证书以及服务端和客户端的公私钥
./easyrsa init-pki													//创建PKI目录用于存储证书
./easyrsa build-ca													//创建根证书，并设置密码用于server和client证书签名
./easyrsa gen-req server nopass							//创建server端的证书和私钥文件
./easyrsa sign server server								//给server端证书签名,确认请求时一定要输入yes，再输CA密码
./easyrsa gen-dh														//创建Diffie-Hellman文件，指定密钥交换时的算法，需要约1-2分钟
./easyrsa gen-req client nopass							//创建client端证书和私钥
./easyrsa sign client client								//给client端证书签名，输入CA密码


#安装openvpn
yum install -y openvpn
#配置openvpn
cp /usr/share/doc/openvpn-2.4.8/sample/sample-config-files/server.conf /etc/openvpn
再去掉注释行和空行改改就成：sed -i -c -e  '/^$/d;/^#/d' server.conf	

cat>/etc/openvpn/server.conf<<EOF
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0							//给客户端分配的地址池，不能与vpn内网一致
push "route 172.16.1.0 255.255.255.0"			//允许客户端访问内网172.16.1.0网段
ifconfig-pool-persist ipp.txt							//地址池记录文件位置
keepalive 10 120													//10秒ping一次，120秒未响应视为断线
max-clients 100
verb 3
client-to-client
log /var/log/openvpn.log
persist-key
persist-tun
duplicate-cn
EOF

把证书公私钥拷贝过来
cp /opt/easy-rsa/pki/ca.crt  /etc/openvpn
cp /opt/easy-rsa/pki/issued/server.crt  /etc/openvpn 
cp /opt/easy-rsa/pki/private/server.key  /etc/openvpn
cp /opt/easy-rsa/pki/dh.pem  /etc/openvpn

开启内核转发
echo "net.ipv4.ip_forward=1">>/etc/sysctl.conf
systemctl restart network
sysctl -p

启动openvpn
systemctl -f enable openvpn@server.service
systemctl start openvpn@server.service
```

##### 客户端连接vpn

```bash
#windows电脑连接
1.下载openvpn客户端软件后，把服务端上/opt/easy-rsa/pki下的密钥文件（ca.crt client.crt client.key）
下载到C:\Program Files\OpenVPN\config目录下

2.在config目录下创建一个客户端配置文件，名：client.ovpn,内容如下
client
dev tun
proto udp
remote 10.0.0.3 1194
resolv-retry infinite
nobind
ca ca.crt
cert client.crt
key client.key
verb 3
persist-key
persist-tun
3.运行桌面上的openvpn拨号连接

#mac本连接vpn
1.

#在linux系统拨vpn
1.安装 yum install -y openvpn
2.拷贝证书和公称钥
3.配置文件
cat >/etc/openvpn/server.ovpn<EOF
		...内容与windows的一致...
EOF
4.启动:
openvpn --daemon --cd /etc/openvpn --config client.ovpn --log-append /var/log/openvpn.log
```

##### 使用vpn访问内网网段

```bash
用户通过vpn拨号，可以通过内网网段ping通vpn服务器，但无法直达web集群
在web01上抓包分析,结果web01的内网网卡有数据包进来，但出不去
tcpdump -i eth1 icmp

解决办法1：在web01添加一条路由，把数据包转给vpn的内网ip，由vpn服务器交给公网ip再回复给用户
route add -net 10.8.0.0/24 gw 172.16.1.3
但这样所有内网主机都要添加一条路由，很麻烦，而且重启就失效，所有采用方法2

解决办法2：在vpn服务器上配置防火墙转发规则
systemctl start firewalld
firewall-cmd --add-service=openvpn  --permanent
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload

```

##### openvpn双重认证

```bash
此时所有用户都是用的同一个证书和密钥，万一离职了要吊销证书那么所有用户都用不了；还有用户只要在电脑上配置了证书和密钥，不需要认证就可以拨vpn连内网，别人用这个电脑就能直接用，不安全

#解决办法：
开启双重认证，新用户加入的时候，添加一个用户名和密码，离职了只要删这个用户名就行了
1.改server.conf配置文件，开启用户名密码验证且允许调用脚本
cat>>/etc/openvpn/server.conf<<EOF
script-security 3																						//允许使用自定义脚本
auth-user-pass-verify /etc/openvpn/check.sh via-env					//指定脚本路径
username-as-common-name																			//开启用户密码登陆验证
EOF
2.写存放用户名密码的文件
cat >>/etc/openvpn/userfile<<EOF
zhouxy zhou1234
EOF
3.使用脚本从文件调取用户名密码，并给脚本加执行权限
cat>/etc/openvpn/check.sh<<EOF
脚本内容见文末
EOF
chmod +x /etc/openvpn/check.sh
4.重启openvpn服务
systemctl restart openvpn@server
5.客户端配置文件也要加用户名密码认证
C:\Program Files\OpenVPN\config\client.ovpn文件加入一条：
auth-user-pass

#使用
当有新员工来就把名字和密码写入userfile
查看添加结果：
tailf /var/log/openvpn/openvpn-password.log
```

##### 脚本文件

```bash
#!/bin/sh
###########################################################
# checkpsw.sh (C) 2004 Mathias Sundman <mathias@openvpn.se>
#
# This script will authenticate OpenVPN users against
# a plain text file. The passfile should simply contain
# one row per user with the username first followed by
# one or more space(s) or tab(s) and then the password.
PASSFILE="/etc/openvpn/userfile"
LOG_FILE="/var/log/openvpn-password.log"
TIME_STAMP=`date "+%Y-%m-%d %T"`
###########################################################
if [ ! -r "${PASSFILE}" ]; then
  echo "${TIME_STAMP}: Could not open password file \"${PASSFILE}\" for reading." >> ${LOG_FILE}
  exit 1
fi
CORRECT_PASSWORD=`awk '!/^;/&&!/^#/&&$1=="'${username}'"{print $2;exit}' ${PASSFILE}`
if [ "${CORRECT_PASSWORD}" = "" ]; then 
  echo "${TIME_STAMP}: User does not exist: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
  exit 1
fi
if [ "${password}" = "${CORRECT_PASSWORD}" ]; then 
  echo "${TIME_STAMP}: Successful authentication: username=\"${username}\"." >> ${LOG_FILE}
  exit 0
fi
echo "${TIME_STAMP}: Incorrect password: username=\"${username}\", password=\"${password}\"." >> ${LOG_FILE}
exit 1
```

























