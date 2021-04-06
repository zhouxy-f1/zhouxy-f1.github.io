## 一、Ansible安装配置

##### 简介

```bash
Ansible是python中的一套模块，使用ssh协议连接来做系统管理及自动化执行命令等任务。ansible很轻量，但由于是串行的，主机超10台会比较慢。大规模自动化需要用saltstack，saltstack是C/S架构，有salt-master和salt-minion，底层是zero-MQ，并行的，比ansible快

#ansible的架构
connectior plugins: 连接插件，用于连接被控端
core modules:	核心模块，用于对被控端实现操作，依赖于具体的模块
custom modules: 自定义模块，可根据自己的需求编写具体的模块
plugins: 插件，对模块功能的补充
playbook: 剧本，是对任务的编排，控制任务执行的先后顺序和依赖关系
inventor: 主机清单，用于定义执行任务的主机有哪些

#ansible执行命令的过程
1.加载配置文件，默认是/etc/ansible/ansible.cfg
2.加载对应的模块
3.将模块生成对应的临时py文件，并把该文件传到远程主机的对应用户的家目录下
4.给远程主机上的.py文件加执行权限
5.执行并返回结果
6.删除临时py文件，退出
```

##### 安装配置

```bash
直接通过yum安装，安装后不需要启动服务就可以直接用，被控端不需要安装ansible。默认配置文件不要改，去工作目录新建即可
通过yum安装的ansible默认是2.9.9版本，底层python2.7.5

#安装ansible
yum install -y ansible									#如果报错404等，可yum cleanall
ansible --version												#安装完成后，查看版本
rpm -ql ansible|grep '^/etc'						#查看默认配置文件
/etc/ansible/ansible.cfg
/etc/ansible/hosts
/etc/ansible/roles

#配置ansible
ansible的配置文件可以有很多，程序会按预先设定的顺序读取配置文件,存在则直接用，建议用~/.ansible.cfg,这样不会污染环境
zcat /usr/share/man/man1/ansible-config.1.gz	#查看配置文件读取顺序
1 $ANSIBLE_CONFIG															#环境变量中定义的配置项,比如export ANSIBLE_SUDO_USER=root
2 ./ansible.cfg																#当前ansible的工作目录下
3 ~/.ansible.cfg															#当前跑ansible的用户家目录下
4 /etc/ansible/ansible.cfg										#系统默认配置文件

配置文件详解（以系统默认配置文件为例）
/etc/ansible/ansible.cfg
[defaults]
inventory      = /etc/ansible/hosts      #主机列表配置文件
library        = /usr/share/my_modules/  #库文件存放目录
remote_tmp     = ~/.ansible/tmp          #临时py文件存放在远程主机目录
local_tmp      = ~/.ansible/tmp          #本机的临时执行目录
forks          = 5                       #默认并发数
sudo_user      = root                    #默认sudo用户
ask_sudo_pass  = True                    #每次执行是否询问sudo的ssh密码
ask_pass       = True                    #每次执行是否询问ssh密码
remote_port    = 22                      #远程主机端口
host_key_checking = False                #跳过检查本机的known_hosts文件中是否有远程主机的指纹
log_path = /var/log/ansible.log          #ansible日志
[privilege_escalation]									 #普通用户提权操作
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```

## 二、Ansible命令

##### 主程序

```bash
/usr/bin/ansible
/usr/bin/ansible-doc
/usr/bin/ansible-galaxy
/usr/bin/ansible-playbook
/usr/bin/ansible-vault
/usr/bin/ansible-console
```

##### 主机清单

```bash
ansible是通过ssh连接被控端，连接方式可通过密码认证和密钥认证，其认证信息是通过/etc/ansible/hosts主机资产清单文件实现，该文件定义了被管理主机的认证信息如ssh登陆用户名、密码、端口、主机名等信息，注意ansible2.0后host、port、user可省略，详情如下：
ansible_ssh_host=10.0.20.92					#指定远程主机
ansible_ssh_port=40022							#指定ssh通过哪个端口连接被控端，不指定的话默认是22
ansible_ssh_user=ncadmin						#指定ssh通过哪个用户连接被控端，不指定的话默认是root
ansible_ssh_pass=xxx								#指定ssh密码，极不安全
ansible_ssh_private_key_file=id_rsa	#ssh使用的私钥文件，当本机有多套密钥时使用

#通过密码认证并测试
把所有用户的密码写在配置文件里极不安全，可在命令行用参数-k(--ask-pass)单独输入密码,但-k只能输入一个密码，所以连接多台主机建议用ssh密钥
vim ./hosts
[web_group]
web1 ansible_host=10.0.20.92 ansible_usr=root ansible_port=22
web2 ansible_host=10.0.20.93 ansible_user=ncamdin ansible_port=40022 ansible_ssh_pass=zmN3SfdF
ansible all -i hosts  -m ping -k	#all是特殊组,指清单中所有主机,-i指定清单,-m指定模块 -k使用密码连接时需要密码

#通过密钥认证
当连接多台被控端时，需要使用密钥认证，具体做法是在本机生成密钥对，再把公钥推送到各被控端,首次连接需要验证密码
当本机有多套密钥时，需要用ansible_ssh_private_key_file指定私钥。

1.在主控端生成密钥对并推送公钥到所有被控端
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub  10.0.20.92									#验证密码，默认推给root用户，使用端口22
ssh-copy-id -i ~/.ssh/in_rsa.pub ncadmin@10.0.20.93 -p 40022	#当hosts里指定别的用户登陆时，公钥推给该用户

2.配置hosts清单
vim ~/ansible/hosts
[web_group]
web1 ansible_host=10.0.20.92 
web2 ansible_host=10.0.20.93 ansible_user=ncadmin ansible_port=40022

3.测试连通性
ansible all -i hosts  -m ping

#关于别名和主机组
定义主机别名和主机组有利于在执行ansible命令时指定相应的主机。别名写在最左边，可以随意命名，一般用主机名，在执行ansible命令选择被控主机时可用配置符和正则表达式；主机组一般用于定义同一集群内的不同主机，如webservers、dbservers等，主机组可以嵌套主机组，如下：
[omega:children]
omega_webs
omega_dbs
[omega_webs]
host1
host2
[omega_dbs]
host1
host2

ansible 'web*' --list-host									#可使用正则来过滤主机别名,?表单字符 *表任意字符
ansible omega --list-host										#默认显示主机别名，无别名时显示主机名,可直接查嵌套在最外层的主机组
ansible longines:&tissot --list-host				#选择多个组时，用或与非，:或 :&与 :!非
ansible all --list-host											#查看所有特殊组all下所有主机，默认读取/etc/ansible/hosts
ansible all -i ansible/hosts  --list-host		#指定hosts文件，查看all组下所有主机
```

##### ad-hoc命令

```bash
ansible命令有两种模式，ad-hoc和playbook。ad-hoc是临时命令，不支持变量，执行完就结束了无法重复使用，但执行效率比playbook高；playbook是对ad-hoc的编排，功能更全面，可以很好的控制命令执行的先后顺序、解决依赖关系。

#ad-hoc命令
ansible --version
ansible webs -m shell -a 'hostname' -i hosts	#-m指定模块名,默认是command,-a指定要执行的命令,-i指定主机清单
ansible webs -m copy -a 'src=/etc/passwd dest=/tmp/passwd'	#把本地文件推送到各个被控端
ansible aa -a 'hostname' -i hosts	-k					#aa是hosts里主机的别名 -k是通过密码验证时要求ssh连接密码
ansible webs -a 'hostname' -C -T 30						#-C模拟执行，不会真的执行; -T执行命令的超时时间,默认10秒
ansible webs -a 'hostname' -v									#-v显示详细信息，-vv -vvv更详细，用于调试
```

##### ad-hoc常用模块

```bash
#ad-hoc查看帮助
ansible-doc -l																#查看支持的模块,-l是--list 可用/copy搜索包含copy的相关模块 
ansible-doc copy	-s													#查看copy模块帮助信息,-s是显示简要信息

#命令模块_command、shell、script: 用于在远程主机执行命令或脚本
command模块：用于执行shell命令，ansible默认模块，但不支持管道等特殊字符
shell模块：用于执行shell命令，用-m shell指定模块，支持特殊字符，但不支持用$符号
script模块：用于执行shell脚本，默认的相对路径是当前工作目录
示例：
ansible all -i hosts -a 'ifconfig'
ansible all -i hosts -m shell -a 'ifconfig|grep eth0'
ansible all -i hosts -m script -a "./diskfree.sh"

#文件管理_copy模块：用于推送本地文件到远程主机，源文件必须存在，目标路径必须存在，否则直接报错
当目标文件重名时，会覆盖掉原文件，需要用backup=yes来备份之前的文件，文件名为srv.txt.9385.2020-06-01@11:43:57~
当需要在新文件写入内容时用content，目标文件存在时覆盖内容，目录文件不存在时自动创建
当需要对拷贝到远程主机的文件指定属主属组权限时，用owner、group、mode，不指定默认root。如果远程主机无该用户则报错
copy也可用于拷贝目录，直接放到dest指定的目录下，目录不存在时自动创建
示例：
ansible all -i hosts -m copy -a 'src=/etc/passwd dest=/opt/srv.txt backup=yes'
ansible all -i hosts -m copy -a "content='jinxiaoqin' dest=/opt/zhouxy.txt backup=yes"
ansible all -i hosts -m copy -a 'src=/etc/passwd dest=/opt/pwd.txt owner=zhou group=zhou mode=2644'

#文件管理_file模块：用于建立目录或文件，并给目录或文件授权
用path指定在哪创建，state=directory时创建目录，state=touch时创建文件;创建时可指定属主属组权限,当对目录递归授权时，如果递归授权时用了mode，则目录下所有文件的权限也会改成755；state=link是创建软链接，state=hard是创建硬链接，此时要用src指定源文件，path指定目标位置；删除文件或目录，path指定目录就删目录，指定文件就删文件,state都是absent
示例：
ansible all -i hosts  -m file -a 'path=/opt/dir state=directory owner=zhou group=zhou mode=777'
ansible all -i hosts  -m file -a 'path=/opt/dir/a.txt state=touch owner=root group=zhou mode=777'
ansible all -i hosts  -m file -a 'src=/opt/p.txt path=/tmp/p.txt state=link'
ansible all -i hosts  -m file -a 'path=/opt/dir owner=zhou group=zhou recurse=yes'
ansible all -i hosts  -m file -a 'path=/opt/dir state=absent'

#文件管理_get_url模块：用于下载文件或目录等
当用dest只指定了路径时，如果该目录有重名文件则直接覆盖；当用dest指定了文件名则下载并改名，还可以指定属主属组权限
ansible all -i hosts  -m get_url -a 'url=http://repo.service.chinanetcloud.com/script dest=/opt/a.txt mode=755 owner=zhou group=zhou'

#用户管理_user、group模块：用于管理用户和组
新建用户时默认创建一个同名的组，默认会创建家目录，默认shell是/bin/bash;用groups指定附加组
ansible all -i hosts -m group -a 'name=dba gid=1024 state=absent|present(默认)'
ansible all -i hosts -m user -a 'name=jonny uid=1024 groups=ops,dba，dev'
ansible all -i hosts -m user -a 'name=www groups=d1m,d2m,d3m shell=/sbin/nologin create_home=false'

#服务管理_yum、service、firewalld、selinux模块：用于下载安装软件、启停服务等
yum模块:用name=nginx表示用yum安装，一般先用copy模块推送repo和rpm包，再用name指定rpm包路径等同本地yum安装,latest表示安装最新版本，present是安装，absent是卸载
service模块:用于起停服务,enabled=yes是加入开机自启，no是取消开机自启，
firewalld模块: 用于管理防火墙,防火墙的启停用service模块，配置规则用firewalld模块
selinux模块: 用于管理selinux
示例：
ansible all -i hosts -m yum -a 'name=nginx state=present|absent|latest'
ansible all -i hosts -m copy -a 'src=./nginx-1.16.1.rpm dest=/usr/local/src/nginx-1.16.1.rpm'
ansible all -i hosts -m yum -a 'name=/usr/local/src/nginx-1.16.1.rpm state=present'
ansible all -i hosts -m service -a 'name=nginx state=started|stopped|restarted|reloaded enabled=no'
ansible all -i hosts -m service -a 'name=firewalld state=stopped'
ansible all -i hosts -m shell -a 'getenforce'					#先查selinux状态
ansible all -i hosts -m shell -a 'setenforce 0'				#临时关闭selinux
ansible all -i hosts -m selinux -a 'state=disabled'		#永久关闭selinux，仅改了配置文件，重启才生效

#解压缩模块
unarchive:
  src: http://download.redis.io/releases/redis-5.0.8.tar.gz
  dest: /opt
  remote_src: yes						#默认是no，指从ansible本地机器的files目录找文件，解压到远程机器的dest指定目录,yes是解压缩远程主机上的文件

#定时任务_cron模块：用于管理定时任务
name=xxx在以后的ansible版本里是必选项，用来给定时任务命名，显示在注释信息里，通过此名称用state=absent来删除定时任务，没有name则无法删除指定的定时任务；用minute=1,2 hour=*/2 day=3-4 month=* weekday来指定分时日月周(,表示不连续 -表连续 */表示隔多久执行一次 不指定则默认是*)；state默认是present，absent是删除定时任务
示例：
ansible all -i hosts -m cron -a 'name=health_check minute=0,5 job="/bin/bash /opt/a.sh"'
ansible all -i hosts -m cron -a 'name=health_check state=absent'

#磁盘挂载_mount模块：用于挂载设备
mounted:挂载设备，并写入配置文件
present:开机挂载，仅写入/etc/fatab配置文件
unmounted:卸载设备，但不清理配置文件,需要配合absent来清理配置
absent:卸载设备，仅清理该设备在/etc/fstab文件的配置

ansible all -i hosts -m mount -a 'src=192.168.20.120:/EWP path=/data fstype=nfs opts=defaults state=mounted|unmounted|absent|present'

#主机信息模块_setup
示例：
ansible all -i hosts  -m setup
ansible all -i hosts  -m setup -a 'filter=ansible_default_ipv4'		#获取ip
ansible all -i hosts  -m setup -a 'filter=ansible_fqdn' 					#获取主机名

ansible_all_ipv4_addresses					仅显示ipv4的信息
ansible_devices											仅显示磁盘设备信息
ansible_distribution								显示是什么系统，例：centos,suse等
ansible_distribution_major_version	显示是系统主版本
ansible_distribution_version				仅显示系统版本
ansible_machine											显示系统类型，例：32位，还是64位
ansible_eth0												仅显示eth0的信息
ansible_hostname										仅显示主机名
ansible_kernel											仅显示内核版本
ansible_lvm													显示lvm相关信息
ansible_memtotal_mb									显示系统总内存
ansible_memfree_mb									显示可⽤系统内存
ansible_memory_mb										详细显示内存情况
ansible_swaptotal_mb								显示总的swap内存
ansible_swapfree_mb									显示swap内存的可⽤内存
ansible_mounts											显示系统磁盘挂载情况
ansible_processor										显示cpu个数(具体显示每个cpu的型号)
ansible_processor_vcpus							显示cpu个数(只显示总的个数)
```

##### playbook

```yml
playbook是除ad-hoc外的另一种命令形式，从某种意义上来说是对ad-hoc临时命令的一种编排，虽然执行效率不如ad-hoc，但可重复使用、支持变量、耦合低，可以很好地控制命令执行顺序及依赖关系，适合大型项目使用

#执行流程
1.Ansible读取playbook剧本，剧本中会记录对哪些主机执行哪些任务
2.首先Ansible通过主机清单找到要执行的主机，然后调用具体的模块
3.其次Ansible会通过连接插件连接对应的主机并推送对应的任务列表
4.最后被管理的主机会将Ansible发送过来的任务解析为本地Shell命令执行

#命令
ansible-playbook main.yml --syntax-check 				#先检查语法
ansible-playbook main.yml	-C										#运行测试不是真的执行
ansible-playbook main.yml												#执行剧本

#语法
用缩进表层级关系，每个缩进由两个空格组成，不能用tab
冒号后必有空格，除非以冒号结尾
短横线表示列表项目，短横线后要有空格


---
#这是一个完整的playbook示例
- hosts: all
  remote_user: root
  become: yes
  become_method: sudo
  vars:
    http_port: 80
    max_clients: 200
  tasks:
    - name: Install nginx
      yum: name=nginx stated= present
      notify:
      - restart nginx
  handlers:
    - name: restart nginx
      systemd: name=nginx state= restartd
```

## 三、Ansible变量

##### 变量定义及优先级

```bash
#变量定义方式及优先级
group_vars < host_vars < 主机清单定义 < playbook < vars_file < 命令行定义
以上位置都可以定义ansible的变量，按读取优先级层层覆盖。常规做法是在group_vars里新建all文件来定义所有变量，有特殊变量就用host_vars，如果需要调试可在命令行用 -e 'name=string'来临时指定变量。

1.group_vars定义
在项目目录下新建一个名为group_vars的目录，在该目录下新建用于定义变量的文件，文件名必须是hosts文件里的组名，该文件里的变量只对该主机组生效；也可新建名为all的文件，对所有主机组生效

示例：
vim /root/ansible/group_vars/webs
soft: nginx,httpd
cmds: tree,lsof

2.host_vars定义
在项目目录下新建一个名为host_vars的目录，该目录下新建文件，名称必须是hosts里主机别名，文件里定义的变量仅对该主机生效

示例：
vim /root/ansible/host_vars/srv-jonny-web001
soft: nginx,httpd
cmds: tree,lsof

3.在主机清单中定义:优先级高于group_vars，容易把变量弄乱，不建议用
[webs]
srv-jonny-web001 ansible_host=10.0.20.92
srv-jonny-web002 ansible_host=10.0.20.95
[webs:vars]
soft=nginx,httpd
cmds=lsof,tree

4.在playbook中定义：仅对当前playbook生效，无法复用，不可取
注意调用多个变量必须用列表形式，但凡在列表里调用的变量都需要加双引号

- hosts: all
  vars:
    - soft: nginx,httpd
    - cmds: tree,lsof
  tasks:
    - name: yum install sofeware and commands
      yum:
        name:
          - "{{soft}}"
          - "{{cmds}}"
        state: absent
    - name: start servers
      service: name= "{{soft}}" state=stopped

5.在vars_file中定义
在项目目录下新建一个yml文件用来定义变量，在main.yml中用vars_files来调用

vim /root/ansible/var1.yml
soft: httpd
cmds: lsof,tree

- hosts: all
  vars_files: 
    - ./var1.yml
    - ./var2.yml
...

6.在命令行定义变量：优先级最高，一般用于调试时用

示例：
ansible-playbook main.yml  -e 'soft=httpd'


#层级定义变量
以上变量如果存在层级关系，可以用层级方式定义，具体定义及调用方法如下：
vim ~/ansible/group_vars/all
lnmp:
  soft:
    web: nginx
    db: tree
    
vim ~/ansible/main.yml

- hosts: all
  tasks:
    - name: install web
      yum:
        name: "{{lnmp.soft.web}}"
    - name: install db
      yum:
        name: "{{lnmp.soft.db}}"
```

##### 变量注册

```bash
ansible的模块在执行完后都会返回结果，但是默认情况是不会显示出来，可以把返回结果存到变量，再调用变量查看返回结果
如下：debug模块用于调试，msg参数用于打印消息，可直接输出变量，默认会输出该变量的所有内容，可用变量属性过滤出想要的

- hosts: all
  tasks:
    - name: yum install sofeware and commands
      shell: ls -l /
      register: list
    - name: return result of ls
      debug:
        msg: "{{list.stdout_lines}}"

---
#1.Update the REPO files as the source of the Ali cloud
#2.Update all packages

- name: check if repos exists
  shell: ls
  args:
    chdir: /etc/yum.repos.d
  register: FileList
  
- name: delete repos
  file:
    path: /etc/yum.repos.d/{{item}}
    state: absent
  with_items:
    - "{{ FileList.stdout_lines }}"
```

##### fact缓存

```bash
ansible默认执行的第一个任务是Gathering Facts，用于采集被控端的主机信息，如主机名、ip、系统版本、cpu、内存、磁盘等
常用于根据不同主机信息定义不同的配置文件，如zabbix配置文件中的主机名，mysql中的最大内存、nginx的cpu亲和等

#默认开启，也可以关闭
- hosts: all
  gather_facts: yes|no
  tasks:
    - name: check info

#以nginx的cpu亲和为例
1.准备配置文件
vim ~/ansible/templates/nginx.conf 
...
user nginx;
worker_processes {{ansible_processor_vcpus}};
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
...

2.推送到远程主机
vim ~/ansible/main.yml
---
- hosts: all
  gather_facts: yes
  tasks:
    - name: copy nginx.conf
      template:
        src: ./templates/nginx.conf
        dest: /etc/nginx/nignx.conf
```

## 四、流程控制

##### 条件

```bash
当需要根据条件还决定是否执行某个任务时，可用条件when来判断，ansible支持的条件运算符有： == != > < >= <= and or not等，也可用括号()对条件进程分组，用列表表示多条件相当于and

#根据主机名做不同操作
- hosts: all
  tasks:
    - name: show info
      debug:
        msg: "{{ansible_distribution_version}}"
      when:
        - ansible_distribution_version|int >6
        - ansible_distribution != "CentOS"
        - ansible_hostname is match "*web*"								#本条件未执行成功
        
#判断文件是否存在
- hosts: all
  vars:
    foopath: /etc/selinux/config
  tasks:
    - debug:
      msg: "file not exist"
      when: foopath is not exists
      when: foopath is exists      
      when: foopath is file
      when: foopath is directory
      when: foopath is link
      when: foopath is mount
      
#根据命令执行状态做判断
- hosts: all
  tasks:
    - shell: "cat /etc/abc"
      register: ReturnMsg
    - debug:
      msg: "success"
      when: ReturnMsg is success
```

##### 循环

```bash
大批量重复性执行可用循环，比如新建100个文件或安装一堆基础软件啥的

#变量循环
- hosts: all
  tasks:
    - name: install a list of package
      yum: name= "{{item}}" state=present
      with_items:
        - httpd
        - tree
        - lsof
        - lszrz
      when: ansible_hostname == "ali-web1"
    - name: show info
      debug: msg="the hostname is {{ansible_hostname}}"
      
#字典循环
- hosts: all
  tasks:
    - name: add users
      group:
        name: "{{item.name}}"
        gid: "{{item.gid}}"
        state: present
      with_items:
        - {name: dev ,gid: 1525}
        - {name: ops ,gid: 1626}
        - {name: dev ,gid: 1727}
```

##### 触发器

```bash
触发器handlers是另一种任务列表，在所有tasks执行完成之后，按照main.yml中handlers排列的顺序依次执行被notify的任务
在特定条件发触发动作，比如修改了nginx配置文件自动重启，通过在修改配置文件的模块设置notify来通知handlers

1.handlers仅会在所有tasks结束后执行一次，不管期间有多少个task通知了handlers；
2.如果不想等所有task执行完了再调用handlers，可在指定的task后加一行-meta: flush_handlers,表示立即调用handlers，等所有task执行完后，再依次调用其它handlers
3.如果设置了notify的模块执行失败，视为没有通知，强制执行handlers的话要用 force-handlers: yes ，与tasks平级
4.如果想用一个notify来调用多个handlers，那么可以用listen来定义一个handlers组，示例如下

- hosts: web1
  force_handlers: yes
  tasks:
    - name: install nginx and php
      yum:
        name: nginx
        state: present
    - name: config nginx
      template:
        src: ./templates/nginx.conf
        dest: /etc/ngin/nginx.conf
      notify:
        - restart nginx
        - meta: flush_handlers
    -meta: flush_handlers
  handlers:
    - name: restart nginx
      listen: restart webserver
      systemd:
        name: nginx
        state: restarted

    - name: restart php-fpm
      listen: restart webserver
      systemd:  
        name: php-fpm
        state: restarted
```

##### 异常处理

```bash
只有当任务状态为changed时，才能通过notify来触发handlers，ansible对changed状态判断不准，需要通过changd_when来修

- hosts: all
  tasks:
    - name: check  nginx
      shell: /usr/sbin/nginx -t
      register: NginxStatus
      failed_when: "'FAILED' in nginx_status.stderr"				#当结果中有FAILED时判定为失败,终止playbook
      changed_when: NginxStatus.stdout.find('successful')	#检查成功则判定为changed，触发重启handlers
      changed_when: false																		#永久不会判定为changed状态
      ignore_errors: yes																		#忽略错误，执行下一个任务
```

##### 任务标签

```bash
默认情况下，ansible会从前到后执行所有的任务，如果只想执行指定的任务，可以给该任务打标签，跑剧本时用-t来指定标签。可以给一个任务定义多个标签，这样用-t来跑剧本时会执行多个任务。

示例：
  tasks:
    - name: install nginx and php
      yum:
        name: nginx
        state: present
      tags: 
        - ngx
        - web
ansible-playbook main.yml  -t ngx
```

##### 文件复用

```bash
在项目目录下新建一个tasks目录，把tasks下拆分好的任务include到main.yml中去

- hosts: all
  remote_user: root
  become: yes
  become_method: sudo
  tasks:
    - include: tasks/base.yml
    - include: tasks/users.yml
    - include: tasks/zabbix.yml
    - include: tasks/packets.yml
```

## 五、jinja2模板

##### 概述

```bash
jinja2模板一般用于根据不同的远程主机信息，渲染不同的配置文件。在项目目录下新建一个templates的目录，templates目录下存放配置文件，其文件名以.j2结尾，在推送配置文件时，用template模块，其作用跟copy模块一样，不同的是template会解析文件里的变量，而copy不会。
注意：jinja2的变量依赖facts缓存，用gather_facts: no后jinja2是不能用的。

#定义
cat roles/Redis/templates/redis.service.j2
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/redis-server  /etc/redis/6379.conf --supervised systemd
ExecStop=/usr/local/bin/redis-cli -h  {{ ansible_facts['default_ipv4']['address'] }} -p 6379 shutdown
Type=forking

[Install]
WantedBy=multi-user.target

#调用
- name: scp template/redis.service.j2 remote_host:/usr/lib/systemd/system/redis.service
  template:
    src: redis.service.j2
    dest: /usr/lib/systemd/system/redis.service

#高级用高
1.判断
{% if ansible_hostname == "web01" %}
        state MASER
        priority 150
{%elif ansible_hostname == "web02"%}
        state BACKUP
        priority 100
{%endif%}

2.循环
upstream {{server_name}} {
        {% for i in range(1,10) %}
        server 172.16.1.{{i}}:{{nginx_port}};
        {% endfor %}
}
```

##### 定义与调用

```bash
#定义模板
upstream {{server_name}} {
        {% for i in range(1,10) %}
        server 172.16.1.{{i}}:{{nginx_port}};
        {% endfor %}
}
server{
        listen {{nginx_port}};
        server_name {{server_name}};
        location / {
                proxy_pass http://{{server_name}};
                proxy_set_header Host $http_host;
        }
}

#调用
- hosts: web_group
  vars:
    - nginx_port: 80
    - server_name: www.123.com
  tasks:
    - name: scp nginx config
      template: src=./www.123.com.conf.j2 dest=~/www.123.com.conf
```

## 六、roles

##### 目录框架

```yml
把所有task写入同一个剧本显然不行，一般使用roles来拆分各工作模块。roles主要依赖目录的命名和路径，调用关系是：
工程目录ProjectName下的abc.yml开始，先读group_vars下变量，再到roles里找tasks的main.yml来执行

#roles目录框架
ProjectName
├── main_entry.yml
├── group_vars
│   └── all
└── roles
    ├── common										#角色名称
    │   ├── defaults
    │   ├── meta									#处理依赖关系
    │		│		└── main.yml
    │   ├── templates							#存放模板
    │		│		└── main.yml
    │   ├── handlers							#触发器
    │		│		└── main.yml
    │   ├── vars									#定义变量
    │		│		└── main.yml
    │ 	├──	files									#存放需要传到远程主机的文件
    │   │   ├── ipvs.modules
    │   │   └── kubernetes.conf
    │   └── tasks									#本角色要执行的任务
    │       ├── main.yml
    │       ├── install_soft.yml
    │       ├── modify_config.yml
    │       └── upgrade_kernel.yml
    ├── dbserver
    │   ├──...
    └── ...

#调用role

vim playbook.yml
---

- hosts: all
  romote_user: root
  pre_tasks:
    - name: done before roles
      shell: echo 'hello'
  roles:
    - {role: Base, when: "ansible_os_family == CentOS"}
    - {role: glusterfs, when: ""}
  tasks:
    - name: xx
  post_tasks:
    - name: do it after roles

ansible-playbook main.yml --limit dbserver
```

##### 其它命令

```bash
#ansible-galaxy
ansible-galaxy search k8s									#查看别人写好的剧本
ansible-galaxy info afonsog.k8s_base			#查看剧本信息
ansible-galaxy install afonsog.k8s_base		#下载剧本到本地
ansible-galaxy init role_name			 				#初始化剧本框架文件

#Ansible-vault
ansible-vault encrypt foo.yml							#加密
ansible-vault view foo.yml								#查看
ansible-vault edit foo.yml								#编辑加密文件
ansible-vault rekey foo.yml								#修改密码
ansible-playbook foo.yml --vault-password-file=ansible.pass	#执行
```

















