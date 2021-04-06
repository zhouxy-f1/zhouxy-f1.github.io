## 理论基础

Ansible是一个自动化统一配置管理工具，

## Ansible与Saltstack对比

```shell
ansible:轻量级，串行。主机超过10台的大规模环境下只通过ssh会很慢，底层是python2，将来有新的bug官方也不再维护

saltstack:使用C/S结构的模式，salt-master和salt-minion，并行的，10台主机以上比ansible快。底层使用的是`zero-MQ`消协队列,saltstack底层既有python2也有python3
```

## Ansible的功能及优点

```shell
1.远程执行
批量执行远程命令，可以对多台主机进行远程操作

2.配置管理
批量配置软件服务，可以进行自动化方式配置，服务的统一配置管理，和启停

3.事件驱动
通过Ansible的模块，对服务进行不同的事件驱动
比如：
1）修改配置后重启
2）只修改配置文件，不重启
3）修改配置文件后，重新加载
4）远程启停服务管理

4.管理公有云
通过API接口的方式管理公有云，不过这方面做的不如`saltstack`.
saltstack本身可以通过saltcloud管理各大云厂商的云平台。

5.二次开发
因为语法是Python，所以便于运维进行二次开发。

6.任务编排
可以通过playbook的方式来统一管理服务，并且可以使用一条命令，实现一套架构的部署

7.跨平台，跨系统
几乎不受到平台和系统的限制，比如安装`apache`和启动服务
在Ubuntu上安装apache服务名字叫apache2
在CentOS上安装apache服务名字叫httpd
在CentOS6上启动服务器使用命令：/etc/init.d/nginx start
在CentOS7上启动服务器使用命令：systemctl start nginx

```

## Ansible的执行流程

```shell
1.Ansible读取playbook剧本，剧本中会记录对哪些主机执行哪些任务。
2.首先Ansible通过主机清单找到要执行的主机，然后调用具体的模块。
3.其次Ansible会通过连接插件连接对应的主机并推送对应的任务列表。
4.最后被管理的主机会将Ansible发送过来的任务解析为本地Shell命令执行。
```

## 安装Ansible

1.环境准备

| 主机名 | wanIP     | lanIP       | 角色          |
| ------ | --------- | ----------- | ------------- |
| m01    | 10.0.0.61 | 172.16.1.61 | Ansible控制端 |
| web01  | 10.0.0.7  | 172.16.1.7  | Ansible被控端 |
| web02  | 10.0.0.8  | 172.16.1.8  | Ansible被控端 |

2.安装ansible

```bash
[root@m01 ~]# yum install -y ansible
```

3.查看ansible模块及版本

```bash
[root@m01 ~]# ansible --version
ansible 2.8.4
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Oct 30 2018, 23:45:53) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

4.ansible参数

```bash
# ansible <host-pattern> [options]
--version   #ansible版本信息
-v          #显示详细信息
-i          #指定主机清单文件路径，默认是在/etc/ansible/hosts
-m          #使用的模块名称，默认使用command模块
-a          #使用的模块参数，模块的具体动作
-k          #提示输入ssh密码，而不使用基于ssh的密钥认证
-C          #模拟执行测试，但不会真的执行
-T          #执行命令的超时
```

5.ansible配置文件读取顺序

```bash
[root@m01 ~]# vim /etc/ansible/ansible.cfg
# nearly all parameters can be overridden in ansible-playbook
# or with command line flags. ansible will read ANSIBLE_CONFIG,
# ansible.cfg in the current working directory, .ansible.cfg in
# the home directory or /etc/ansible/ansible.cfg, whichever it
# finds first

1、$ANSIBLE_CONFIG
2、./ansible.cfg
3、~/.ansible.cfg
4、/etc/ansible/ansible.cfg
```

## 配置文件

```bash
#inventory      = /etc/ansible/hosts      #主机列表配置文件
#library        = /usr/share/my_modules/  #库文件存放目录
#remote_tmp     = ~/.ansible/tmp          #临时py文件存放在远程主机目录
#local_tmp      = ~/.ansible/tmp          #本机的临时执行目录
#forks          = 5                       #默认并发数
#sudo_user      = root                    #默认sudo用户
#ask_sudo_pass = True                     #每次执行是否询问sudo的ssh密码
#ask_pass      = True                     #每次执行是否询问ssh密码
#remote_port    = 22                      #远程主机端口
host_key_checking = False                 #跳过检查主机指纹
log_path = /var/log/ansible.log           #ansible日志

#普通用户提权操作
[privilege_escalation]
#become=True
#become_method=sudo
#become_user=root
#become_ask_pass=False 
```

## ansible Inventory（主机清单文件）

**场景一：密码方式连接**

```bash
[root@m01 ~]# cat /etc/ansible/hosts

#方式一、IP+端口+用户+密码
[webs]
10.0.0.7 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='1'
10.0.0.8 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='1'

#方式二、主机名+密码
[webs]
web0[1:2] ansible_ssh_pass='123456'

#方式三、主机+密码
[webs]
web0[1:2]
[webs:vars]
ansible_ssh_pass='123456'

注意：方式二和方式三，都需要做hosts解析
```

**场景二：密钥方式连接**

```bash
#创建密钥对
[root@m01 ~]# ssh-keygen
#推送公钥
[root@m01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.16.1.7
[root@m01 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.16.1.8

#方式一：
[web_group]
172.16.1.7
172.16.1.8

#方式二：
[webs]
web01 ansible_ssh_host=172.16.1.7
web02 ansible_ssh_host=172.16.1.8
```

**场景三：主机组定义方式**

```bash
[web_group]
web01 ansible_ssh_host=172.16.1.7
web02 ansible_ssh_host=172.16.1.8

[db_group]
db01 ansible_ssh_host=172.16.1.51
lb01 ansible_ssh_host=172.16.1.5
[db_group:vars]
ansible_ssh_pass='1'

[nfs_group]
nfs ansible_ssh_host=172.16.1.31

[nfs_server:children]
web_group
nfs_group

[lnmp:children]
web_group
db_group


[root@m01 ~]# ansible 'all' --list-host
  hosts (5):
    nfs
    web01
    web02
    db01
    lb01
[root@m01 ~]# ansible 'web_group' --list-host
  hosts (2):
    web01
    web02
[root@m01 ~]# ansible 'db_group' --list-host
  hosts (2):
    db01
    lb01
[root@m01 ~]# ansible 'lnmp' --list-host
  hosts (4):
    web01
    web02
    db01
    lb01
```

## ad-hoc模式命令使用

结果返回颜色

```bash
绿色: 代表被管理端主机没有被修改
黄色: 代表被管理端主机发现变更
红色: 代表出现了故障，注意查看提示
```

## ansible常用模块

##### 1.command

```bash
[root@m01 ~]# ansible 'web_group' -m command -a 'free -m'
web02 | CHANGED | rc=0 >>
              total        used        free      shared  buff/cache   available
Mem:            972         140         489           7         342         658
Swap:          1023           0        1023

web01 | CHANGED | rc=0 >>
              total        used        free      shared  buff/cache   available
Mem:            972         113         412          13         446         669
Swap:          1023           0        1023
```

##### 2.shell

```bash
[root@m01 ~]# ansible 'web_group' -m shell -a 'ps -ef|grep nginx'
web02 | CHANGED | rc=0 >>
root      12584  12583  0 20:16 pts/1    00:00:00 /bin/sh -c ps -ef|grep nginx
root      12586  12584  0 20:16 pts/1    00:00:00 grep nginx

web01 | CHANGED | rc=0 >>
root      14575  14570  0 12:16 pts/1    00:00:00 /bin/sh -c ps -ef|grep nginx
root      14577  14575  0 12:16 pts/1    00:00:00 grep nginx
```

##### command与shell区别

1）command不支持特殊符号

2）shell模块支持特殊符号

3）不指定-m 默认使用的是command模块

##### 3.script

可以在远程主机上执行本地脚本或命令

```shell
[root@mgr ~]# vim 1.sh 
#!/usr/bin/bash
mkdir /root/xx

[root@mgr ~]# ansible web01 -m script -a '/root/1.sh'
web01 | CHANGED => {
    "changed": true, 
    "rc": 0, 
    "stderr": "Shared connection to 172.16.1.7 closed.\r\n", 
    "stderr_lines": [
        "Shared connection to 172.16.1.7 closed."
    ], 
    "stdout": "", 
    "stdout_lines": []
}
```

##### 4.yum

实现软件或服务的安装、删除等，可安装yum源文件或本地rpm包

```bash
用法：
name                            
		包名:                        #指定要安装的软件包名称
    file://                     #指定本地安装路径（yum localinstall 本地rpm包）
    http://                     #指定yum源（从远程仓库获取rpm包）
    
state                           #指定使用yum的方法
    installed,present           #安装软件包
    removed,absent              #移除软件包
    latest                      #安装最新软件包

1)安装服务，相当于：yum install -y vsftpd
[root@m01 ~]# ansible 'web_group' -m yum -a 'name=vsftpd state=present'

2)通过url安装服务，相当于 yum install -y url=***
[root@m01 ~]# ansible 'web_group' -m yum -a 'name=https://mirrors.aliyun.com/zabbix/zabbix/4.0/rhel/7/x86_64/zabbix-agent-4.0.0-2.el7.x86_64.rpm state=present' 

3)用yum安装本地rpm包，相当于yum localinstall -y *.rpm
[root@m01 ~]# ansible 'web_group' -m yum -a 'name=file:///root/nagios-4.4.3-1.el7.x86_64.rpm state=present'

4、删除软件或服务，相当于：yum remove -y vsftpd
[root@m01 ~]# ansible 'web_group' -m yum -a 'name=vsftpd state=absent'

```

##### 5.yum_repository

对repo文件的增删改等操作

```bash
1、用法
name 							#指定仓库名字
file							#指定仓库文件名,当未指定file时，会用仓库名字给文件命令
description				#添加描述（repo文件中的name）
baseurl						#指定yum仓库的地址
gpgcheck					#是否开启校验
		yes
		no
enabled						#是否启用yum仓库
		yes
		no
state
		absent				#删除yum仓库
		present				#创建yum仓库（默认）


#添加yum仓库
ansible 'web_group' -m yum_repository -a 'name=zls_epel,zls_base  description=EPEL baseurl=https://download.fedoraproject.org/pub/epel/$releasever/$basearch/ gpgcheck=no enabled=yes file=zls_epel'

#添加mirrorlist
ansible 'web_group' -m yum_repository -a 'name=zls_epel  description=EPEL baseurl=https://download.fedoraproject.org/pub/epel/$releasever/$basearch/ gpgcheck=no enabled=yes file=epel mirrorlist=http://mirrorlist.repoforge.org/el7/mirrors-rpmforge'

#删除yum仓库
ansible 'web_group' -m yum_repository -a 'name=zls_epel file=zls_epel state=absent'

#修改yum仓库
ansible 'web_group' -m yum_repository -a 'name=epel  description=EPEL baseurl=https://download.fedoraproject.org/pub/epel/$releasever/$basearch/ gpgcheck=no enabled=no file=epel'

```

## 文件管理模块

##### 1.copy

把本地文件推送到远程主机，还可以在远程主机新建文件、命令、写入内容等

```bash
src					#指定推送的源文件
dest				#指定推送的目标位置
owner				#指定属主
group				#指定属组
mode				#指定权限（数字方式）
content				#在指定文件中添加内容
backup				#是否备份（注意：控制端和被控端，内容不一致才会备份）
	yes
	no
	
- name: Copy file with owner and permissions
  copy:
    src: /srv/myfiles/foo.conf
    dest: /etc/foo.conf
    owner: foo
    group: foo
    mode: '0644'


#推送文件
[root@m01 ~]# ansible 'web_group' -m copy -a 'src=/root/index.html dest=/var/www/html owner=root group=root mode=0644'

#推送文件并备份
[root@m01 ~]# ansible 'web_group' -m copy -a 'src=/root/index.html dest=/var/www/html owner=root group=root mode=0644 backup=yes'

#编辑nfs配置文件
[root@m01 ~]# ansible 'web_group' -m copy -a 'content="/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)" dest=/etc/exports'


```

##### 2.file

创建、删除文件或目录，同时指定权限、属主属组等，还可以做软链接

```bash
src							#指定软链接的源文件
dest						#指定软连接的目标文件
path						#指定创建目录或文件
state
		touch				#创建文件
		directory		#创建目录
		absent			#删除目录或文件
		link				#做软链接
owner						#指定属主
group						#指定属组
mode						#指定权限
recurse					#递归授权
		yes
		no

#创建目录 mkdir
[root@m01 ~]# ansible 'web_group' -m file -a 'path=/backup state=directory owner=adm group=adm mode=0700'

#创建多级目录并递归授权chown -R  chmod -R
[root@m01 ~]# ansible 'web_group' -m file -a 'path=/zls/mysql/db01 state=directory owner=adm group=adm mode=0700 recurse=yes'

#创建文件（前提条件，上级目录必须存在）  touch
[root@m01 ~]# ansible 'web_group' -m file -a 'path=/root/zls.txt state=touch'

#删除目录  rm -fr
[root@m01 ~]# ansible 'web_group' -m file -a 'path=/backup state=absent'

#做软链接 ln -s
[root@mgr ~]# ansible web01 -m file -a 'src=/root/zxy.txt dest=/root/zxy.txt.lk state=link'
```

##### 3.get_url

```bash
url					#指定下载文件的url
dest				#指定下载的位置
mode				#指定下载后的权限
checksum		#校验
	md5				#md5校验
	sha256		#sha256校验- name: Download foo.conf


#下载worldpress代码
[root@m01 ~]# ansible 'web_group' -m get_url -a 'url=http://test.driverzeng.com/Nginx_Code/wordpress-5.0.3-zh_CN.tar.gz dest=/root mode=0777'

#下载并校验MD5
[root@m01 ~]# ansible 'web_group' -m get_url -a 'url=http://test.driverzeng.com/Nginx_Code/test.txt dest=/root mode=0777 checksum=md5:ba1f2511fc30423bdbb183fe33f3dd0f'

#下载repo文件
[root@mgr ~]# ansible web01 -m get_url -a 'url=http://mirrors.aliyun.com/repo/epel-7.repo dest=/etc/yum.repos.d/epel.repo'
```

## 服务管理模块

##### 1.service,systemd

```bash
name					#指定服务名称
state
	started				#启动
	stopped				#停止
	restarted			#重启
	reloaded			#重载
enabled					#是否开机自启
	yes
	no
	
[root@m01 ~]# ansible 'web_group' -m systemd -a 'name=httpd state=stopped enabled=yes'
[root@m01 ~]# ansible 'web_group' -m systemd -a 'name=httpd state=started enabled=yes'
[root@m01 ~]# ansible 'web_group' -m systemd -a 'name=httpd state=restarted enabled=yes'
[root@m01 ~]# ansible 'web_group' -m systemd -a 'name=httpd state=reloaded enabled=yes'
```

------

## 用户管理模块

##### 1.group

```bash
name				#指定组名
gid					#指定gid
state
	present		#创建
	absent		#删除

#创建组
[root@m01 ~]# ansible 'web_group' -m group -a 'name=www gid=666 state=present'

#删除组
[root@m01 ~]# ansible 'web_group' -m group -a 'name=www gid=666 state=absent'
```

##### 2.user

```bash
name								#指定用户名
uid									#指定uid
group								#指定属组
groups							#指定附加组
state
	present						#创建用户
	absent						#删除用户
shell								#指定用户登录的shell
	/bin/bash
	/sbin/nologin
create_home					#是否创建家目录
	true
	false
comment							#添加注释
generate_ssh_key		#创建密钥对
ssh_key_bits				#指定密钥对长度
ssh_key_file				#指定密钥文件

#创建用户
[root@m01 ~]# ansible 'web_group' -m user -a 'name=www uid=666 group=www state=present shell=/sbin/nologin create_home=false'

#删除用户
[root@m01 ~]# ansible 'web_group' -m user -a 'name=www uid=666  state=absent'

#创建用户的同时创建密钥对
[root@m01 ~]# ansible 'web_group' -m user -a 'name=zls generate_ssh_key=yes ssh_key_bits=2048 ssh_key_file=.ssh/id_rsa'
```

## 作业：主机清单及部署rsync服务

##### 1.全部主机清单inventory

```shell
[root@mgr ~]# cat /etc/ansible/hosts 
[lb_group]
lb01 ansible_ssh_host=172.16.1.5
lb02 ansible_ssh_host=172.16.1.6

[web_group]
web01 ansible_ssh_host=172.16.1.7
web02 ansible_ssh_host=172.16.1.8

[nfs_group]
nfs ansible_ssh_host=172.16.1.31

[backup_group]
backup ansible_ssh_host=172.16.1.41

[db_group]
db01 ansible_ssh_host=172.16.1.51
db02 ansible_ssh_host=172.16.1.52

[zabbix_group]
zabbix ansible_ssh_host=172.16.1.71

[rsync_server:children]
backup_group
web_group

[nfs_server:children]
nfs_group
web_group
```

##### 2.部署httpd php  mariadb 

1)在web_group安装httpd和php-fpm

```shell
[root@mgr ~]# ansible web_group -m yum -a 'name=httpd state=present'
[root@mgr ~]# ansible web_group -m yum -a 'name=php-fpm state=present'
```

2)在db_group安装mariadb

```shell
[root@mgr ~]# ansible db_group -m yum -a 'name=mariadb-server state=present'
```

3)上传作业代码

```shell
[root@mgr ~]# ansible web_group -m shell -a 'cd /var/www/html && rz'
```

##### 3.一键部署rsync 服务端和客户端（backup，web01，web02）

##### 1)安装rsync

需要在backup、web01、web02上安装

```shell
[root@mgr ~]# ansible rsync_server -m yum -a 'name=rsync state=present'
```

##### 2)配置文件

```shell
[root@mgr ~]# vim rsyncd.conf 
uid=www
gid=www
port=873
fake super=yes
use chroot=no
max connetcion=200
timeout=600
ignore errors
read only=false
list=false
auth users=rsync_backup
secrets file=/etc/rsync.passwd
log file = /var/log/rsyncd.log

[zhouxy]
comment=commit
path=/backup

[root@mgr ~]# ansible backup_group -m copy -a 'src=/root/rsyncd.conf dest=/etc/'
```

##### 3)创建www用户

```shell
[root@mgr ~]# ansible rsync_server -m group -a 'name=www gid=666 state=present'
[root@mgr ~]# ansible 'web_group' -m user -a 'name=www uid=666 group=www state=present shell=/sbin/nologin create_home=false'
```

##### 4)创建密码文件并授权

```shell
[root@mgr ~]# ansible backup_group -m copy -a 'content="rsync_backup:123456" dest=/etc/rsync.passwd mode=0600'
[root@mgr ~]# ansible web_group -m copy -a 'content="123456" dest=/etc/rsync.pass mode=0600'
```

##### 5)创建/backup目录

```shell
[root@mgr ~]# ansible backup_group -m file -a 'path=/backup state=directory owner=www group=www '
```

##### 6)启动服务

```shell
[root@mgr ~]# ansible rsycn_server -m systemd -a 'name=rsyncd state=started enabled=yes'
```

##### 7)一键部署

```shell
[root@mgr ~]#vim ansible.sh

#!/bin/bash
ansible rsync_server -m yum -a 'name=rsync state=present' &&/
ansible backup_group -m copy -a 'src=/root/rsyncd.conf dest=/etc/' &&/
ansible rsync_server -m group -a 'name=www gid=666 state=present' &&/
ansible 'web_group' -m user -a 'name=www uid=666 group=www state=present shell=/sbin/nologin create_home=false' &&/
ansible backup_group -m copy -a 'content="rsync_backup:123456" dest=/etc/rsync.passwd mode=0600' &&/
ansible web_group -m copy -a 'content="123456" dest=/etc/rsync.pass mode=0600' &&/
ansible backup_group -m file -a 'path=/backup state=directory owner=www group=www ' &&/
ansible rsycn_server -m systemd -a 'name=rsyncd state=started enabled=yes' &&/
```

