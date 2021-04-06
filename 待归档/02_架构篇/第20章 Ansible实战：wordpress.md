## 课题：部署wordpress站点

在web01和web02上安装nginx和php，部署好站点目录/code/wordpress后用远程数据库连接，在web上写文章，导出数据库备用，再把原wordpress站点重新打包，解压。

## 环境准备

| 主机名 |   内网IP    | 角色                  | 需要安装的服务             |
| :----: | :---------: | --------------------- | :------------------------- |
| web01  | 172.16.1.7  | web服务器             | nginx ，php-fpm，nfs-utils |
| web02  | 172.16.1.8  | web服务器             | nginx，php-fpm，nfs-utils  |
|  nfs   | 172.16.1.31 | nfs服务器，备用backup | nfs-utils，sersync         |
| backup | 172.16.1.41 | 备份服务器            | rsync                      |
|  db01  | 172.16.1.51 | 数据库服务器          | mariadb-server             |
|  mgr   | 172.16.1.61 | 管理机                | ansible                    |

## 一、项目规划

##### 1.安装ansible

```shell
[root@mgr ~]# yum install -y ansible
```

##### 2.项目规划

```shell
[root@m01 ~]# tree project/
project/
├── base
│   ├── base.yml
│   ├── CentOS-Base.repo
│   ├── epel.repo
│   ├── nginx.repo
│   └── php.repo
├── mariadb
│   ├── dump_mysql.yml
│   ├── mariadb.yml
│   └── wordpress.sql
├── nfs
│   └── nfs.yml
├── rsync
│   ├── backup.sh
│   ├── rsyncd.conf
│   └── rsyncd.yml
├── sersync
│   ├── sersync2.5.4_64bit_binary_stable_final.tar.gz
│   └── sersync.yml
└── web
    ├── lnmp.yml
    ├── nginx-1.16.1.tar.gz
    ├── nginx.conf
    ├── nginx.service
    ├── php71w-7.1.31.tgz
    ├── wordpress.tgz
    ├── www.123.com.conf
    └── www.conf

6 directories, 22 files
```

##### 3.主机清单

```shell
[root@mgr ~]# vim /etc/ansible/hosts
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

[rsync_server:children]
backup_group
nfs_group

[nfs_server:children]
nfs_group
web_group

```

##### 4.分发公钥

```shell
[root@mgr ~]# ssh-keygen
[root@mgr ~]# ssh-copy-id  -i ~/.ssh/id_rsa.pub 172.16.1.7
[root@mgr ~]# ssh-copy-id  -i ~/.ssh/id_rsa.pub 172.16.1.8
[root@mgr ~]# ssh-copy-id  -i ~/.ssh/id_rsa.pub 172.16.1.31
[root@mgr ~]# ssh-copy-id  -i ~/.ssh/id_rsa.pub 172.16.1.41
[root@mgr ~]# ssh-copy-id  -i ~/.ssh/id_rsa.pub 172.16.1.51
```

## 二、Base基础优化

```shell
[root@mgr /project/base]# vim base.yml
- hosts: all
  tasks:
    - name: stop firewalld
      systemd:
        name: firewalld
        state: stopped
    - name: create group www
      group:
        name: www
        gid: 666
    - name: creat user www
      user:
        name: www
        uid: 666
        group: www
        shell: /sbin/nologin
        create_home: false
    - name: install unzip
      yum:
        name: unzip
        state: present
    - name: Disable SElinux
      selinux:
        state: disabled
    - name: copy yum repo
      copy:
        src: /etc/yum.repos.d/CentOS-Base.repo
        dest: /etc/yum.repos.d/
        ower: root
        group: root
        mode: 0644
```

## 三、部署web

##### 1.在web目录需要准备的文件

```shell
[root@m01 ~/project/web]# ll
total 37M
-rw-r--r--. 1 root root 1.8K Sep 22 03:02 lnmp.yml
-rw-r--r--. 1 root root 1009K Sep 19 11:46 nginx-1.16.1.tar.gz   /nginx源码包
-rw-r--r--. 1 root root 2.7K Sep 21 23:38 nginx.conf
-rw-r--r--. 1 root root  415 Sep 21 22:22 nginx.service
-rw-r--r--. 1 root root  19M Aug 23 06:34 php71w-7.1.31.tgz
-rw-r--r--. 1 root root  18M Sep 22 02:44 wordpress.tgz		   /在web服务器上写完文章后把wordpress目录打包
-rw-r--r--. 1 root root  250 Sep 20 15:51 www.123.com.conf   
-rw-r--r--. 1 root root  18K Sep 20 15:59 www.conf
```

##### 2.一键部署

```shell
[root@m01 ~/project/web]# vim lnmp.yml 
- hosts: web01
  tasks:
    - name: Unarchive php.tar
      unarchive:
        src: ./php71w-7.1.31.tgz
        dest: /root
    - name: Install php
      shell: "cd ~/php71w-7.1.31 && rpm -Uvh *.rpm"   
    - name: config php
      copy:
        src: ./www.conf
        dest: /etc/php-fpm.d/
 
    - name: Unarchive nginx tar
      unarchive:
        src: ./nginx-1.16.1.tar.gz
        dest: /root
        copy: yes 
    - name: Install nginx request packages
      yum:
        name:
          - pcre-devel
          - openssl-devel
          - zlib-devel
        state: present
    - name: Install nginx
      shell: "cd ~/nginx-1.16.1 && ./configure --prefix=/usr/local/nginx --user=www --group=www --with-http_ssl_module --with-http_stub_status_module && make && make install"
    - name: creat dir conf.d
      file:
        path: /usr/local/nginx/conf/conf.d
        state: directory        
    - name: Scp nginx.service
      copy:
        src: ./nginx.service
        dest: /usr/lib/systemd/system/nginx.service
    - name: Scp nginx.conf
      copy:
        src: ./nginx.conf
        dest: /usr/local/nginx/conf/nginx.conf
    - name: Scp www.123.com.conf
      copy:
        src: ./www.123.com.conf
        dest: /usr/local/nginx/conf/conf.d/    
    - name: mkdir /code
      file:
        path: /code
        state: directory
    - name: unarchive wordpress
      unarchive:
        src: ./wordpress-5.2.3-zh_CN.zip
        dest: /code
        owner: www
        group: www
    - name: make soft_link
      file:
        src: /usr/local/nginx/sbin/nginx
        dest: /usr/sbin/nginx
        state: link
   
    - name: Start php-fpm
      systemd:
         name: php-fpm
         state: started
         enabled: yes
    - name: Start nginx
      systemd:
         name: nginx
         state: started
         enabled: yes
```

##### 3.执行结果

![image-20190922171211430](https://tva1.sinaimg.cn/large/006y8mN6gy1g78etlwo7fj313y0b3wf7.jpg)

## 四、部署数据库

##### 1.在mariadb目录准备的文件

```shell
[root@m01 ~/project/mariadb]# ll
total 496K
-rw-r--r--. 1 root root  143 Sep 22 02:35 dump_mysql.yml
-rw-r--r--. 1 root root  837 Sep 22 02:45 mariadb.yml
-rw-r--r--. 1 root root 488K Sep 22 02:46 wordpress.sql

```

##### 2.一键部署代码

```shell
[root@m01 ~/project/mariadb]# vim mariadb.yml 
- hosts: db_group
  tasks:
    - name: Install  MySQL-python
      yum:
        name: MySQL-python
        state: present
    - name: Install mariadb server
      yum:
        name: mariadb-server
        state: present
    - name: Start mysql
      systemd:
        name: mariadb
        state: started
        enabled: yes
    - name: Create database wordpress
      mysql_db:
        name: wordpress
        state: present
    - name: Create user of wordpress
      mysql_user:
        priv: "*.*:All"
        name: "wp"
        host: 172.16.1.%
        password: "123"
        state: present
    - name: Scp mysql data
      copy:
        src: ./wordpress.sql
        dest: /tmp/wordpress.sql
    - name: Import data of sql to wordpress
      mysql_db:
        state: import
        name: all
        target: /tmp/wordpress.sql

```

#####    3.执行结果

![image-20190922030109967](https://tva1.sinaimg.cn/large/006y8mN6gy1g77q82ggnhj313z0k0n4j.jpg)



## 五、部署NFS

##### 1.部署代码

```shell
[root@m01 ~/project/nfs]# vim nfs.yml 
- hosts: nfs_server
  tasks:
    - name: Install nfs-utils
      yum:
        name: nfs-utils
        state: present

- hosts: nfs_group
  tasks:
    - name: config /etc/exports
      copy:
        content: "/data 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)"
        dest: /etc/exports
    - name: Create /data/wordpress
      file:
        path: /data/wordpress
        state: directory
        owner: www
        group: www
    - name: Scp uploads to nfs
      copy:
        src: ~/project/web/wordpress/wp-content/uploads/
        dest: /data/wordpress/
    - name: Start nfs server
      systemd:
        name: nfs-server
        state: started
        enabled: yes
- hosts: web_group
  tasks:
    - name: Mount web to nfs
      mount:
        path: /code/wordpress/wp-content/uploads/
        src: 172.16.1.31:/data/wordpress
        fstype: nfs
        state: mounted

```

##### 2.执行结果

```shell
Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.8 www.123.com
Macbook:~ zhouxy$ ping www.123.com
PING www.123.com (10.0.0.8): 56 data bytes
64 bytes from 10.0.0.8: icmp_seq=0 ttl=64 time=0.412 ms

```

## 六、部署rsync

##### 1.准备文件

```shell
[root@m01 ~/project/rsync]# ll
total 12K
-rw-r--r--. 1 root root  570 Sep 20 16:24 backup.sh
-rw-r--r--. 1 root root  463 Sep 22 18:49 rsyncd.conf
-rw-r--r--. 1 root root 1.2K Sep 22 19:06 rsyncd.yml
```

##### 2.部署代码

```shell
[root@m01 ~/project/rsync]# vim rsyncd.yml 
- hosts: rsync_server
  tasks:
    - name: Install rsyncd server
      yum:
        name: rsync
        state: present
- hosts: backup_group
  tasks:
    - name: Config rsyncd conf
      copy:
        src: ~/project/rsync/rsyncd.conf
        dest: /etc/rsyncd.conf
        owner: root
        group: root
        mode: 0644
    - name: Create /backup on nfs
      file:
        path: /backup
        state: directory
        owner: www
        group: www
        mode: 0755
        recurse: yes
    - name: Create passwd file
      copy:
        content: "rsync_back:123"
        dest: /etc/rsync.passwd
        owner: root
        group: root
        mode: 0600
    - name: Start rsyncd
      systemd:
        name: rsyncd
        state: started
        enabled: yes
- hosts: web_group
  tasks:
    - name: Config client passwd file
      copy:
        content: "123"
        dest: /etc/rsync.pass
        owner: root
        group: root
        mode: 0600
    - name: Copy shell
      copy:
        src: ./backup.sh
        dest: /root
    - name: Add to crontab
      cron:
        name: "backup"
        minute: "00"
        hour: "01"
        job: "/bin/sh/root/backup.sh &>/dev/null"

```

## 七、部署sersync

##### 1.准备文件

```shell
[root@m01 sersync]# ll
total 716
-rwxr-xr-x 1 root root   2216 Sep 18 17:47 confxml.xml
-rw-r--r-- 1 root root 727290 Aug  7 11:40 sersync2.5.4_64bit_binary_stable_final.tar.gz
-rw-r--r-- 1 root root      0 Sep 18 17:44 sersync.yml
```

##### 2.部署代码

```shell
[root@m01 ~/project/sersync]# vim sersync.yml 
- hosts: nfs_group
  tasks:
    - name: Install inotify
      yum:
        name: {{ pack }}
      vars:
        pack:
          - rsync
          - inotify-tools

    - name: Config client passwd file
      copy:
        content: "123"
        dest: /etc/rsync.pass
        owner: root
        group: root
        mode: 0600

    - name: jieya
      unarchive:
        src: ./sersync2.5.4_64bit_binary_stable_final.tar.gz
        creates: /usr/loca/sersync
        copy: yes

    - name: copy config file
      copy:
        src: ./confxml.xml
        dest: /usr/local/GNU-Linux-x86
        owner: root
        group: root
        mode: 0755
 
    - name: start sersync
      shell: "/usr/local/GNU-Linux-x86/sersync2 -rdo /usr/local/GNU-Linux-x86/confxml.xml"
```





















