## 条件语句

可以用于检测软件包是否安装，如果未安装就用yum模块安装，已安装就跳过；也可以根据web服务器不同的操作系统安装不同的软件，比如 CentOS 装httpd ，在Ubantu上装apache2

##### 判断操作系统

如果web服务器是CentOS系统，安装http服务用httpd包，如果是Ubuntu，则用httpd2包

```shell
[root@m01 ~]# vim when.yml
- hosts: web_group
  tasks:
    - name: Install httpd server
      yum: name=httpd state=present
      when: ansible_distribution=="CentOS"
    - name: Install httdp server
      apt: name=httpd2 state=present
      when: ansible_distribution=="Ubuntu"
```

##### 判断主机名

为所有主机名为web的服务器添加nginx仓库，其余都跳过

```shell
[root@m01 ~]# cat when2.yml 
- hosts: all
  tasks:
    - name: Add nginx repo
      yum_repository:
        name: nginx_t
        description: Nginx yum repo
        baseurl: http://nginx.org/packages/centos/7/$basearch/
        gpgcheck: no
      when: (ansible_hostname is match "web*") or (ansible_hostname is match "lb*")
```

##### 判断服务是否启动



##### 重启服务前检查语法

task执行nginx -t 后状态自动为changed，会触发重启。可以使用changed_when，nginx -t的返回值中查找到successful字符时才返回changed状态，再触发重启。如果nginx -t 出现失败，可用ignore_errors: yes 忽略报错

遗留问题：nginx -t 检测到语法错误时会报错，tasks终止执行，如何不提醒报错

```shell
[root@m01 ~]# vim check.yml 
- hosts: web_group
  tasks:
    - name: Scp nginx.conf & conf.d
      template: src={{ item.src }} dest={{item.dest}}
      with_items:
        - {src: "./nginx.conf.j2", dest: "/usr/local/nginx/conf/nginx.conf"}
        - {src: "./www.123.com.conf.j2", dest: "/usr/local/nginx/conf/conf.d/www.123.com.conf"}
    - name: Check syntax
      shell: "/usr/sbin/nginx -t"
      register: check_result
      changed_when: check_result.stdout.find('successful')
      notify: syntax_ok_restart
  handlers:
    - name: syntax_ok_restart
      systemd: name=nginx state=restarted
```

## 常规循环

用systemd模块一次性nginx和php-fpm服务，实现方式是：所有主机启动nginx服务完成后，所有主机再启php-fpm

##### 批量启动服务

```shell
[root@m01 ~]# vim item.yml 
- hosts: web_group
  tasks:
    - name: Start nginx and php server
      systemd:
        name: "{{ item }}"
        state: started
      with_items:
        - nginx
        - php-fpm

```

## 变量循环

##### 批量安装软件包

```shell
#成功          
[root@m01 ~]# cat item.yml 
- hosts: web_group
  tasks:
    - name: Install a list of packages
      yum:
        name: "{{ packages }}" 
        state: present
      vars:
        packages:
          - tree
          - httpd
          - httpd-tools

#注意：用yum: name=的时候全绿，没成功也没报错 
[root@m01 ~]# cat item.yml 
- hosts: web_group
  tasks:
    - name: Install a list of packages
      yum: name= "{{ packages }}" state=present
      vars:
        packages:
          - tree
          - httpd
          - httpd-tools

```

## 字典循环

当需要对一组变量指定不同的操作时，需要用字典循环，比如批量拷贝文件时，需要指定不同文件到不同目录，给不同权限

##### 批量创建用户

```shell
#创建用户依然需要先创建组
- hosts: web_group
  tasks:
    - name: Add Users
      user: name={{ item.name }} group={{ item.group }} state=present
      with_items:
        - {name: 'testuser1' , group: 'bin'}
        - {name: 'testuser1' , group: 'root'}
```

```shell
- hosts: web_group
  tasks:
    - name: Add Users
      user:
        name: "{{ item.name }}"
        group: "{{ item.group }}"
        state: present
      with_items:
        - {name: 'testuser1' , group: 'bin'}
        - {name: 'testuser1' , group: 'root'}
```

##### 批量拷贝文件

```shell
[root@m01 ~]# cat with.yml 
- hosts: web_group
  tasks:
    - name: scp rsyncd.conf & rsync.passwd
      template: src={{ item.src }} dest={{ item.dest }} mode={{ item.mode }}
      with_items:
        - { src: "./rsyncd.conf", dest: "/etc/rsyncd.conf", mode: "0644"} 
        - { src: "./rsync.pass", dest: "/tmp/rsync.pass", mode: "0600"} 
        
        #冒号后要有空格
```

注：从执行结果可以看出执行顺序，先拷贝rsyncd.conf，全部主机完成后再拷贝rsync.pass

![image-20190924232225905](https://tva1.sinaimg.cn/large/006y8mN6gy1g7b0regme3j30ps036wf7.jpg)



## handlers触发器

当ansible执行task任务后，状态发生了改变，需要触发某个动作时用，比如修改完配置文件需要自动重启服务。

1、handlers触发器是根据changed（黄色）状况触发，执行查看等任务也会改变状态触发handlers，需要强制改为ok

2、handlers与tasks平级，当所有task成功执行了才能触发，且不管有多少notify，handlers只执行一次

##### 当配置文件修改后，重启服务

```shell
#也可以用列表，重启多个服务
[root@m01 ~]# cat handler.yml 
- hosts: web_group
  vars:
    - httpd_port: 8033
  tasks:
    - name: Install httpd server
      yum: name=httpd state=present

    - name: configure httpd
      template: src=./httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf 
      notify:
        - Restart httpd server
        - Restart php server
     
    - name: Start httpd server
      systemd: name=httpd state=started

  handlers:
    - name: Restart httpd server
      systemd: name=httpd state=restarted
    - name: Restart php server
      systemd: name=php-fpm state=restarted
```

##### 执行结果

```shell
[root@m01 ~]# ansible-playbook handler.yml 

PLAY [web_group] **********************************************************************************************
TASK [Gathering Facts] **********************************************************************************************
ok: [web02]
ok: [web01]
TASK [Install httpd server] **********************************************************************************************
ok: [web02]
ok: [web01]
TASK [configure httpd] **********************************************************************************************
changed: [web02]
changed: [web01]
TASK [Start httpd server] **********************************************************************************************
ok: [web01]
ok: [web02]
RUNNING HANDLER [Restart httpd server] *********************************************************************************************
changed: [web01]
changed: [web02]
RUNNING HANDLER [Restart php server] **********************************************************************************************
changed: [web02]
changed: [web01]
PLAY RECAP **********************************************************************************************
web01  : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
web02  : ok=6    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## tags标签

在做playbook调试时，默认按顺序执行所有的任务，效率太低。可用tag标签指定执行个别的任务，或不执行指定的任务

可以对一个task打一个标签，也可以对一个task打多个标签，还可以对多个task打一个标签用来捆绑任务

用-t 执行指定的tag任务；用--skip-tags指定不执行的tag任务

##### 定义tags标签

```shell
[root@m01 ~]# cat tag.yml 
- hosts: web_group
  vars:
    - httpd_port: 8033
  tasks:
    - name: Install httpd server
      yum: name=httpd state=present
      tags:
        - httpd_install
        - httpd_server

    - name: configure httpd
      template: src=./httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf 
      notify: Restart httpd server
      tags: 
        - httpd_config

    - name: Start httpd server
      systemd: name=httpd state=started
      tags:
        - httpd_start
        - httpd_server

  handlers:
    - name: Restart httpd server
      systemd: name=httpd state=restarted
```

##### 查看所有tags

```shell
[root@m01 ~]# ansible-playbook tag.yml  --list-tags
playbook: tag.yml
  play #1 (web_group): web_group        TAGS: []
      TASK TAGS: [httpd_config, httpd_install, httpd_server, httpd_start]
```

##### 只执行指定tags

```shell
#只执行配置相关的任务
[root@m01 ~]# ansible-playbook tag.yml  -t httpd_config 

#执行多个tags，用逗号隔开
[root@m01 ~]# ansible-playbook tag.yml  -t httpd_config，httpd_start
```

##### 排除指定，执行其它

```shell
#排除安装任务，别的都要执行
[root@m01 ~]# ansible-playbook tag.yml  --skip-tags httpd_install
```

## 文件复用

按安装、配置、启动等功能把文件拆分成不同的模块，方便调用及后续维护、调试

##### 任务调用

调用任务时任务文件中只需要有- name任务就行

```shell
#拆分
[root@m01 ~]# vim httpd_install.yml 
- name: Install httpd server
  yum: name=httpd state=present
  
[root@m01 ~]# vim httpd_config.yml        
- name: configure httpd
  template: src=./httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf 
  notify: Restart httpd server
  
[root@m01 ~]# vim httpd_start.yml 
- name: Start httpd server
  systemd: name=httpd state=started

#导入
[root@m01 ~]# vim task_main.yml
- hosts: web_group
  vars:
    - httpd_port: 808
  tasks:
    - include_tasks: httpd_install.yml
    - include_tasks: httpd_config.yml
    - include_tasks: httpd_start.yml
  handlers:
    - name: Restart httpd server
      systemd: name=httpd state=restarted

```

##### 文件调用

调用文件时每个文件都必须是完整的playbook

```shell
[root@m01 ~]# vim main.yml
- import_playbook: handlers.yml
- import_playbook: when.yml

```

## 异常处理

##### 忽略错误

playbook默认依次执行所有的task任务，当遇到执行不成功，则后面的所有任务都停止执行，可在任务下添加忽略错误

```shell
[root@m01 ~]# vim ignore.yml 
- hosts: web_group
  tasks:
    - name: ignore
      command: /bin/false
      ignore_errors: yes
    - name: touch file
      file: path=/tmp/ig.t state=touch
```

##### 强制调用handlers

当修改完配置文件后需要重启服务，如果playbook中任一task执行失败，都会停止执行，则无法触发handlers，也就不能重启，想用ignore_errors忽略错误，又无法预判位置时，需要用到强制调用

```shell
#配置文件执行成功，后面的安装 出错，强制调用handlers来重启服务
[root@m01 ~]# vim err.yml 
- hosts: web_group
  vars:
    - httpd_port: 8035
  force_handlers: yes
  tasks:
    - name: configure httpd
      template: src=./httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf 
      notify: Restart httpd server
     
    - name: Install httpd server
      yum: name=httpdaaa state=present

    - name: Start httpd server
      systemd: name=httpd state=started

  handlers:
    - name: Restart httpd server
      systemd: name=httpd state=restarted
```

##### 强制变绿

用于决定是否触发handlers，当执行查看等命令时，ansible默认状态为changed，黄色的，会触发handlers。此时可用changed_when: false来强制改变状态为ok

```shell
#仅查看远程主机，ansible自动判断状态为changed,就会触发handlers
[root@m01 ~]# cat err1.yml 
- hosts: web_group
  tasks:
    - name: Check syntax
      shell: /usr/sbin/nginx -t
      notify: Change
  handlers:
    - name: Change
      file: path=/tmp/changeddd state=touch
      
#结果是调用了handlers
[root@web01 ~]# ll /tmp
total 12K
-rw-r--r--. 1 root root   0 Sep 25 14:19 changeddd

#解决办法
[root@m01 ~]# cat err1.yml 
- hosts: web_group
  tasks:
    - name: Check syntax
      shell: /usr/sbin/nginx -t
      notify: Change
  handlers:
    - name: Change
      file: path=/tmp/changeddd state=touch
```

## 加密模块

##### 语法

```shell
[root@m01 ~]# ansible-vault [create|decrypt|edit|encrypt|encrypt_string|rekey|view] [vaultfile.yml]

#加密
[root@m01 ~]# ansible-vault encrypt aa.yml 
New Vault password: 
Confirm New Vault password: 
Encryption successful

#解密
[root@m01 ~]# ansible-vault decrypt aa.yml 
Vault password: 
Decryption successful
[root@m01 ~]# 

#改密码
[root@m01 ~]# ansible-vault rekey aa.yml     
Vault password: 
New Vault password: 
Confirm New Vault password: 
Rekey successful

#查看文件内容
[root@m01 ~]# ansible-vault view aa.yml 
Vault password: 
- hosts: web_group
  tasks:
  ....
  
#修改文件内容
[root@m01 ~]# ansible-vault edit aa.yml      
Vault password: 


#执行文件
[root@m01 ~]# echo "123" > ansible.pass
[root@m01 ~]# ansible-playbook aa.yml --vault-password-file=./ansible.pass

```









































