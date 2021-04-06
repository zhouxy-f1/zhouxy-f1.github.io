## 基础语法

##### 判断

```shell
{% if ansible_hostname == "web01" %}
        state MASER
        priority 150
{%elif ansible_hostname == "web02"%}
        state BACKUP
        priority 100
{%endif%}
```

##### 循环

```shell
upstream {{server_name}} {
        {% for i in range(1,10) %}
        server 172.16.1.{{i}}:{{nginx_port}};
        {% endfor %}
}
```

## jinja2语法示例

##### 渲染nginx配置文件

```shell
[root@m01 ~]# vim www.123.com.conf.j2
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
```

```shell
[root@m01 ~]# vim jj.yml 
- hosts: web_group
  vars:
    - nginx_port: 80
    - server_name: www.123.com
  tasks:
    - name: scp nginx config
      template: src=./www.123.com.conf.j2 dest=~/www.123.com.conf
```

##### 渲染keepalived

```shell
[root@m01 ~]# cat keepalived.conf.j2 
global_defs{
        rooter_id {{ansible_hostname}}
}
vrrp_instance VI_1 {
{% if ansible_hostname == "web01" %}
        state MASER
        priority 150
{%elif ansible_hostname == "web02"%}
        state BACKUP
        priority 100
{%endif%}
}
        interface eth0
        virtual_router_id 50
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass 1111
        }
        virtual_ipaddress {
                10.0.0.3
        }

```

```shell
[root@m01 ~]# cat jk.yml 
- hosts: web_group
  tasks:
    - name: Scp keeplived conf
      template: src=./keepalived.conf.j2 dest=/tmp/k.conf
```

##### 渲染mysql端口

```shell
#设置端口用变量，没设置用默认3306
[root@m01 ~]# vim my.cnf.j2 
{% if PORT %}
bind-address=0.0.0.0:{{ PORT }}
{% else %}
bind-address=0.0.0.0:3306
{% endif %}

[root@m01 ~]# vim jmy.yml 
- hosts: web_group
  vars:
    PORT：1290
 #  PORT: false
  tasks:
    - name: Scp my.cnf
      template: src=./my.cnf.j2 dest=/tmp/my.cnf
```

