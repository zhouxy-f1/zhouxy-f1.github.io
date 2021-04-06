## 变量的定义方式

##### 1.使用group_vars,host_vars 

这两个目录存放位置与yml文件在同级目录，group_vars和host_vars下的文件名必须和inventory清单中定义的组名或主机名一致，特殊组 all，在group_vars/all 中定义的变量对所有主机生效，优先级：host_vars  > group_vars

```shell
#定义
[root@m01 ~/project/test]# tree
.
├── group_vars
│   └── web_group
├── host_vars
│   └── web01
└── mk.yml

1.主机组变量：文件名跟主机组名称一致，作用域为本主机组
[root@m01 ~]# vim group_vars/web_group
file_name: XXX

2.主机变量文件：文件名与主机名保持一致，其作用域仅限此主机，优先级高于主机组
[root@m01 ~]# vim host_vars/web01 
file_name: web01

#调用
[root@m01 ~]# vim mkfile.yml 
- hosts: web01
  tasks:
    - name: touch file named web01
      file:
        path: /tmp/ {{file_name}}
        state: touch
- hosts: web_group
  tasks:
    - name: touch  file named web_group
      file:
        path: /tmp/ {{ file_name }}
        state: touch

#执行结果
[root@web01 /tmp]# ll
total 0
-rw-r--r--. 1 root root 0 Sep 24 09:19  web01
```

##### 2.使用vars_files指定变量文件

文件名可以随意，所有playbook可以共用，需要时用vars_files导入即可

```shell
#定义：新建一个或多个文件用于存放变量，文件名随意，规范取名以.j2结尾
[root@m01 ~]# vim vars.yml.j2
file_name: varsfile
[root@m01 ~]# vim var.yml.bak 
name: xxx
#调用:在- hosts下导入vars.yml文件即可
[root@m01 ~]# vim mkf.yml 
- hosts: web01
  vars_files: 
    - ./vars.yml.j2
    - ./vars.yml.bak
    
#执行结果
[root@web01 /tmp]# ll
total 0
-rw-r--r--. 1 root root 0 Sep 24 10:26  varsfile
-rw-r--r--. 1 root root 0 Sep 24 10:26  xxx
```

##### 3.在playbook中直接定义

作用域仅限本playbook

```shell
#定义
[root@m01 ~/project/test]# cat mk.yml 
- hosts: web01
  vars:
    - file_name: pb
    - dir_name: pb_dir
#调用
   tasks:
     - name: touch file web01 in web01
       file:
         path: /tmp/ {{file_name}}
         state: touch
- hosts: web_group
  tasks:
    - name: touch  file web
      file:
        path: /tmp/1/ {{ file_name }}
        state: touch
```

##### 4.在主机清单中定义

变量环境会特别乱，不建议使用，建议在group_vars中定义

```shell
#定义
[root@m01 ~]# vim /etc/ansible/hosts
[rsync_server:vars]
web_pkg=httpd

[web_group:vars]
web_pkg=nginx

#调用
[root@m01 ~]# vim test.yml
- hosts: zls_web01
  tasks:
    - name: Install web server
      yum:
        name: "{{ web_pkg }}"
        state: present

```

##### 5.在命令行定义

优先级最高，使用参数  --extra-vars  或 -e 来定义，一般用于临时测试

```shell
[root@m01 /tmp]# ansible-playbook -e "file_name=abc" test.yml  
```

## 变量优先级

```shell
命令行 > 导入的vars_files > playbook中直接定义的vars > host_vars > group_vars > group_vars/all

#测试方法
[root@m01 project1]# cat p5.yml 
- hosts: webservers
#  vars:
#    filename: play_vars
#  vars_files:
#    - ./vars.yml
  tasks:

    - name: Create 
      shell: mkdir -pv /tmp/{{ filename }}
      register: mk_test

    - name: debug
      debug: msg={{ mk_test }}
```

## 变量层级定义

变量的归属更清晰，但定义和调用都比较麻烦，建议定义变量名时用驼峰定义法，效果一样，更简单

```shell
#定义
[root@m01 /tmp]# vim vars.j2
rainbow:
  web:
    web_package: httpd
    db_package: mariadb
code:
  web:
    file_name: code_web_filename
    
#调用
[root@m01 /tmp]# vim test.yml
- hosts: web01
  vars_files: ./vars.j2
  tasks:
    - name: Create file
      file:
        path: /tmp/ {{ code["web"]["file_name"] }}	
        state: touch
 注：调用时也可以用 code.web.file_name 但playbook中一般不这么用
```

## 注册变量

与模块同级。执行某个任务后在屏幕没有打印结果时，可以用register来获取并存放到自定义的变量，再用debug模块输出变量，可用于判断？

```shell
#使用方法
[root@m01 ~]# vim reg.yml 
- hosts: web01
  tasks:
    - name: Get network status
      shell: netstat -lntup
      register: net_status
    - name: Output status
      debug:
        msg: "当前rc值为：{{ net_status.rc }}"
#执行结果
TASK [Get network status] ***********************************************************************************************
changed: [web01]
TASK [Output status] ***********************************************************************************************
ok: [web01] => {
    "msg": "当前rc值为：0"
}
PLAY RECAP ***********************************************************************************************
web01   : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

## facts变量

用途：可以通过facts变量采集远程主机的信息，如主机名、IP、系统版本、内存状态、磁盘状态等等

```shell
[root@m01 ~]# vim facts.yml 
- hosts: web_group
  tasks:
    - name: Show info of  hostname & IP
      debug:
        msg: >
          the ipv4 of  {{  ansible_fqdn}} is {{ ansible_default_ipv4.address }}

#执行结果
TASK [Show info of  hostname & IP] *************************************************
ok: [web01] => {
    "msg": "the ipv4 of  web01 is 10.0.0.7\n"
}
ok: [web02] => {
    "msg": "the ipv4 of  web02 is 10.0.0.8\n"
}
```

##### 调主机名生成zabbix配置文件

通过facts检查hostname，来生成对应不同主机的zabbix配置文件

```shell
[root@m01 ~]# vim zabbix_agentd.conf
Hostname={{ ansible_hostname }}

[root@m01 ~]# vim facts.yml 
- hosts: web_group
  tasks:
    - name: send zabbix.conf to webs
      template:
        src: ./zabbix_agentd.conf
        dest: /tmp/zabbix_agentd.conf
```

##### 调用内存生成mysql配置文件

通过facts检查内存，按各主机总内存的80%，生成不同的mysql配置文件

```shell
vim my.conf
10.0.0.8
{{ ansible_memtotal_mb *4//5 }}
```

##### facts常用变量

```shell
#获取IP地址
[root@m01 ~]# ansible web01 -m setup -a "filter=ansible_default_ipv4" 
```

```shell
ansible_default_ipv4:仅显示IPv4的信息
ansible_devices:仅显示磁盘设备信息
ansible_distribution:显示操作系统，如CentOS RedHat
ansible_distribution_major_version：显示操作系统的主版本号 如：7
ansible_distribution_version：显示操作系统的版本号 如：7.6
ansible_machine:显示系统是32位还是64位
ansible_eth0：仅显示eth0信息
ansible_hostname：仅显示主机名
ansible_kernel：仅显示内核版本
#ansible_lvm：显示lvm相关信息
ansible_memtotal_mb：显示系统总内存
ansible_memfree_mb：显示系统可用内存
#ansbile_memory_mb：显示系统内存详情
ansible_swaptotal_mb：显示总的swap内存
ansible_swapfree_mb：显示可用的swap内存
ansible_mounts:显示系统磁盘挂载情况
ansible_processor:显示cpu个数及型号
ansible_processor_vcpus:仅显示总的cpu个数
```

##### 关闭facts

采集远程主机内置变量会影响性能，架构达几百台主机时，建议对于不需要用变量的主机可以关闭采集

```shell
[root@m01 ~]# vim facts.yml 
- hosts: web_group
  gather_facts:no
  tasks:
    - name: send zabbix.conf to webs
```







































