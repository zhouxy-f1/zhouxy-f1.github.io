## ssh

##### 远程连接原理

```bash
#简介
ssh是secure shell的缩写，由于是加密传输，常用于远程连接。ssh在服务端的守护进程名为sshd，默认22端口，负责实时监听22端口来自客户端的请求。ssh是C/S架构，服务端由openssh提供ssh服务，其服务名为sshd，包含ssh连接和sftp服务；openssl提供加密服务；客户端openssh-clients，客户端包含ssh连接命令和scp命令

#远程连接加密类型及优缺点
1.对称加密：使用同一套密钥来加密解密，加密强度高但密钥一旦泄漏，全盘垮掉
2.非对称加密：密钥分公钥和私钥，公钥用于加密，私钥来解密。在服务端生成公私钥，客户端请求登陆服务端时把公钥发给客户端，客户端用这个公钥加密自己的密码发给服务端，服务端用私钥解密后验证该密码的合法性，验证通过允许登陆。此时如果有人冒充服务端把自己的公钥发给客户端，再用自己的私钥解密就能拿到客户端的登陆账户和密码，再拿账号和密码去登陆服务端，此为中间人攻击

#ssh如何防止中间人攻击：确保客户端收到的公钥是目标客户端生成的
1.通过密码认证
由于服务端的公私钥都是自己生成的，不像https中可以通过CA来进行公证，所以需要客户端自己确认。
客户端首次连接服务端时会提示：远程主机可靠性未认证，其公钥指纹(RAS公钥1024位，用hash转成128位指纹)是xxx,是否继续？
输入yes后，服务端的hostkey会保存到客户端的~/.ssh/known_host文件中，下次连服务端不用再认证，但每次连接都需要输密码
其登陆过程是：客户端发起连接请求-->服务端发送自己的公钥到客户端-->客户端确认服务端身份(仅首次，使用hostkey)-->输入密码并用该公钥加密发给服务端-->服务端解密并验证身份后返回登陆结果

2.通过密钥认证
每次远程登陆都要输入密码这对脚本、ansible等很不友好，不想验证密码直接登陆的话需要用密钥认证
在客户端用ssh-keygen命令或者用xshell、CRT等工具生成公钥私钥，把公钥拷贝到服务端的~/.ssh/authorized_keys文件中，但第一次登陆还是要验证服务端的hostkey，并加入到客户端的~/.ssh/known_host文件中。
客户端发起登陆请求，服务端生成随机数并将随机数用客户端复制过来的公钥加密后回传给客户端，客户端用私钥解密随机数，再把随机数和session_id用MD5生成摘要发给服务端，服务端把这个摘要跟自己生成的摘要对比，相同则完成认证，允许登陆。

相关概念
hostkey:在服务端生成,用于证明自己身份的非对称密钥,启动sshd服务时自动生成,位于/etc/ssh/ssh_host_rsa_key.pub
authorized_keys: 是在服务端上保存已授权的客户端公钥，通过手动copy或ssh-copy-id命令添加的
RSA DSA ECDSA Ed25519加密


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

#ssh优化
vim /etc/ssh/sshd_config 
Port 1152												#更改端口为1162
PermitRootLogin no							#禁止root用户直接远程登陆（用普通用户登陆再切换到root是可以的）
PasswordAuthentication no				#禁止使用密码直接远程登陆
UseDNS no												#禁止ssh进行DNS反向解析，连接更快

#其它
expect或sshpass来免交互
ssh -o StrictHostKeyChecking=no ncamdin@10.0.20.44
sshpass -p dsADfsdf ssh -o StrictHostKeyChecking=no ncadmin@10.3.1.100 free -h
```

##### ssh配置文件详解

```bash

```

