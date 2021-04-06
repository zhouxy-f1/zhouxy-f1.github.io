## 什么是playbook

playbook（剧本）是由两部分组成

`play`：主机或者主机组（角色：可以有一个或者多个）

`task`：指定工作（动作，台词：一个或者多个）

----

在Ansible中"剧本文件"是以yml结尾的文件。

在SaltStack中"剧本文件"是以sls结尾的文件。

但是语法，使用的都是yaml语法

## playbook和Ad-Hoc对比

| 特点     | PlayBook | ad-hoc |
| -------- | -------- | ------ |
| 完整性   | √        | ✘      |
| 持久性   | √        | ✘      |
| 执行效率 | 低       | 高     |
| 变量     | 支持     | 不支持 |
| 耦合度   | 低       | 高     |

1.`PlayBook`功能比`ad-hoc`更全，是对`ad-hoc`的一种编排.
2.`PlayBook`能很好的控制先后执行顺序, 以及依赖关系.
3.`PlayBook`语法展现更加的直观.
4.`playbook`可以持久使用,`ad-hoc`无法持久使用.



## YAML语法

```shell
#角色
- hosts: web_group

#动作
  tasks:
    - name: install httpd server
      yum:
        name: httpd
        state: present

    - name: start httpd server
      systemd:
        name: httpd
        state: started
        enabled: yes
        
#检查语法
--syntax-check

冒号：只要不是以冒号结尾的冒号，冒号后面都要加空格
短横线-：代表一个层级，在Python中专业叫法，代表是一个列表
```

## 安装httpd练习

##### 1.安装httpd

```bash
[root@m01 httpd]# vim httpd.yml
#角色
- hosts: web_group

#动作
  tasks:
    - name: install httpd server
      yum:
        name: httpd
        state: present

#检查语法
[root@m01 httpd]# ansible-playbook --syntax-check httpd.yml 

playbook: httpd.yml

#执行
[root@m01 httpd]# ansible-playbook  httpd.yml 
PLAY [web_group] ***************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************
ok: [zls_web02]
ok: [zls_web01]

TASK [install httpd server] ***************************************************************************************************
changed: [zls_web01]
changed: [zls_web02]

PLAY RECAP ***************************************************************************************************
zls_web01   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
zls_web02   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

##### 2.启动httpd

```bash
#角色
- hosts: web_group

#动作
  tasks:
    - name: install httpd server
      yum:
        name: httpd
        state: present

    - name: start httpd server
      systemd:
        name: httpd
        state: started
        enabled: yes
```

##### 3.关闭防火墙

```bash
- hosts: web_group

#动作
  tasks:
#关闭防火墙
    - name: Stop Firewalld
      systemd:
        name: firewalld
        state: stopped
        enabled: no

#安装httpd
    - name: install httpd server
      yum:
        name: httpd
        state: present

#开启httpd
    - name: start httpd server
      systemd:
        name: httpd
        state: started
        enabled: yes
```

##### 4.给默认站点页面

```bash
#角色
- hosts: web_group

#动作
  tasks:
#关闭防火墙
    - name: Stop Firewalld
      systemd:
        name: firewalld
        state: stopped
        enabled: no

#安装httpd
    - name: install httpd server
      yum:
        name: httpd
        state: present

#开启httpd
    - name: start httpd server
      systemd:
        name: httpd
        state: started
        enabled: yes

#配置默认页面
    - name: Config index.html
      copy:
        content: "zls_web_page"
        dest: /var/www/html/index.html
        group: root
        owner: root
        mode: 0644
```

##### 5.给不同的web配置不同的页面(多个play)

```bash
#角色
- hosts: web_group

#动作
  tasks:
#关闭防火墙
    - name: Stop Firewalld
      systemd:
        name: firewalld
        state: stopped
        enabled: no

#安装httpd
    - name: install httpd server
      yum:
        name: httpd
        state: present

#开启httpd
    - name: start httpd server
      systemd:
        name: httpd
        state: started
        enabled: yes

- hosts: zls_web01
  tasks:
    - name: Config index.html
      copy:
        content: "zls_web01_page"
        dest: /var/www/html/index.html
        group: root
        owner: root
        mode: 0644

- hosts: zls_web02
  tasks:

    - name: Config index.html
      copy:
        content: "zls_web02_page"
        dest: /var/www/html/index.html
        group: root
        owner: root
        mode: 0644
```

## rsyncd实战

##### 1.环境准备

| 主机名 | wanIP     | lanIP       | 服务        | 角色           |
| ------ | --------- | ----------- | ----------- | -------------- |
| m01    | 10.0.0.61 | 172.16.1.61 | Ansible     | 控制端（导演） |
| backup | 10.0.0.41 | 172.16.1.41 | rsync服务端 | 被控端（男一） |
| web01  | 10.0.0.7  | 172.16.1.7  | rsync客户端 | 被控端（女二） |
| web02  | 10.0.0.8  | 172.16.1.8  | rsync客户端 | 被控端（女二） |

##### 2.战前准备

```bash
#准备项目目录
[root@m01 project]# mkdir rsyncd

#配置文件
uid = www
gid = www
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = false
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
#####################################
[backup]
comment = welcome to oldboyedu backup!
path = /backup

#准备主机清单
[web_group]
zls_web01 ansible_ssh_host=172.16.1.7
zls_web02 ansible_ssh_host=172.16.1.8

[backup_group]
backup ansible_ssh_host=172.16.1.41

[rsync_server:children]
web_group
backup_group
```

##### 3.写剧本

```bash
#1.安装rsync服务端和客户端
#2.配置rsync服务端
#3.创建目录授权
#4.创建密码文件授权
#5.创建系统用户
#6.启动rsync并加入开机自启
```

##### 4.实施

```bash
[root@m01 rsyncd]# vim rsyncd.yml
[root@m01 rsyncd]# cat rsyncd.yml 
- hosts: rsync_server
  tasks:

#关闭防火墙
    - name: Stop firewalld
      systemd:
        name: firewalld
        state: stopped
        enabled: no

    - name: SCP YUM REPO
      copy:
        src: /etc/yum.repos.d/CentOS-Base.repo
        dest: /etc/yum.repos.d/

#安装rsync服务端和客户端
    - name: Install rsyncd Server
      yum:
        name: rsync
        state: present
#创建系统用户组
    - name: Create www Group
      group:
        name: www
        gid: 666
        state: present
#创建系统用户
    - name: Create www User
      user:
        name: www
        uid: 666
        group: www
        create_home: false
        shell: /sbin/nologin


- hosts: backup_group
  tasks:
#配置rsync服务端
    - name: Config rsyncd Conf
      copy:
        src: ./rsyncd.j2
        dest: /etc/rsyncd.conf
        owner: root
        group: root
        mode: 0644

#创建目录授权
    - name: Create dir
      file:
        path: /backup
        state: directory
        owner: www
        group: www
        mode: 0755
        recurse: yes

#创建密码文件授权
    - name: Create passwd file
      copy:
        content: "rsync_backup:123"
        dest: /etc/rsync.passwd
        owner: root
        group: root
        mode: 0600


#启动rsync并加入开机自启
    - name: Start rsyncd
      systemd:
        name: rsyncd
        state: started
        enabled: yes

#配置客户端
- hosts: web_group
  tasks:
    - name: Config client passwd file
      copy:
        content: "123"
        dest: /etc/rsync.pass
        owner: root
        group: root
        mode: 0600
```

