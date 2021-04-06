## 一、安装部署

```bash
#准备java环境
上传jdk包(jdk: java development kit 开发套装)
mkdir /app && tar xf jdk-8u60-linux-x64.tar.gz -C /app
ln -s jdk1.8.0_60/ jdk
cat >>/etc/profile<<'EOF'																												//环境变量
export JAVA_HOME=/opt/software/jdk
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib:$JAVA_HOME/jre/lib:$JAVA_HOME/lib/tools.jar
EOF
. /etc/profile																																	//使环境变量生效

#安装tomcat
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.0.9/bin/apache-tomcat-8.0.9.tar.gz
tar xf apache-tomcat-8.0.27.tar.gz -C /app
/app/tomcat/bin/version.sh 																											//检查是否配置成功
/app/tomcat/bin/startup.sh 																											//启动tomcat
注：/app/tomcat/bin/shutdown.sh																									 //关闭tomcat
注：/app/tomcat/bin/catalina.sh																		//启动关闭tomcat时调用的核心脚本
```

##### 目录结构

```bash
.
├── bin
│   ├── bootstrap.jar
│   ├── catalina.sh
│   ├── catalina-tasks.xml
│   ├── ciphers.sh
│   ├── commons-daemon.jar
│   ├── commons-daemon-native.tar.gz
│   ├── configtest.sh
│   ├── daemon.sh
│   ├── digest.sh
│   ├── setclasspath.sh
│   ├── shutdown.sh
│   ├── startup.sh
│   ├── tomcat-juli.jar
│   ├── tomcat-native.tar.gz
│   ├── tool-wrapper.sh
│   └── version.sh
├── conf
│   ├── Catalina
│   ├── catalina.policy
│   ├── catalina.properties
│   ├── context.xml
│   ├── jaspic-providers.xml
│   ├── jaspic-providers.xsd
│   ├── logging.properties
│   ├── server.xml									#主配置文件
│   ├── tomcat-users.xml						#管理端的配置文件
│   ├── tomcat-users.xsd
│   └── web.xml											#辅助配置文件，用于增加插件或做优化功能
├── lib
│   ├── ***
│   └── websocket-api.jar
├── logs
│   ├── catalina.2021-04-06.log
│   ├── catalina.out
│   ├── host-manager.2021-04-06.log
│   ├── localhost.2021-04-06.log
│   ├── localhost_access_log.2021-04-06.txt	#tomcat的访问日志
│   └── manager.2021-04-06.log
├── temp
│   └── safeToDelete.tmp
├── webapps
│   ├── docs
│   ├── examples
│   ├── host-manager
│   ├── manager
│   └── ROOT
└── work
    └── Catalina
```

##### 详解主配置文件

```bash
路径：/app/tomcat/conf/server.xml
`这是关闭tomcat的管理端口,用法是：本机连接 telnet 127.0.0.1 8005 再输入：SHUTDOWN就能关闭端口，但进程还在
<Server port="8005" shutdown="SHUTDOWN">

`管理端，运维或测试用，一般关闭防止用户登陆																							
<Resource name="UserDatabase" auth="Container"
          type="org.apache.catalina.UserDatabase"
          description="User database that can be updated and saved"
          factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
          pathname="conf/tomcat-users.xml" />				
`控制线程数，默认最大150个，闲时留4个 
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
          maxThreads="150" minSpareThreads="4"/>       

`web端口，默认8080 ，https时8443
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
`当tomcat需要跟apache通信时通过8009端口，AJP是一种工作方式
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />      

`虚拟主机部分：hostname相当于nginx的server_name;appBase就是root；unpackWARs指自动解压WAR包；autoDeploy自动部署到内存；最底下pattern是指日志格式
<Host name="localhost"  appBase="webapps"
      unpackWARs="true" autoDeploy="true">
  <!-- SingleSignOn valve, share authentication between web applications
       Documentation at: /docs/config/valve.html -->
  <!--
  <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
  -->
  <!-- Access log processes all example.
       Documentation at: /docs/config/valve.html
       Note: The pattern used is equivalent to using pattern="common" -->
  <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
         prefix="localhost_access_log" suffix=".txt"
         pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
```

##### 管理控制台

```
管理控制台是tomcat自带的web应用管理平台，默认情况下TomcatManager只能通过localhost:8080访问，需要解除访问的地址限制，再新建管理用户并绑定角色，默认角色有4种分别代表不同的权限

#打开控制台
1.解除访问地址限制，需要注释掉context.xml中<Value>这一整行
vim webapps/manager/META-INF/context.xml
<Context antiResourceLocking="false" privileged="true" >
  <CookieProcessor className="org.apache.tomcat.util.http.Rfc6265CookieProcessor"
                   sameSiteCookies="strict" />
  <!--Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" /-->
  <Manager sessionAttributeValueClassNameFilter="java\.lang\.(?:Boolean|Integer|Long|Number|String)|org\.apache\.catalina\.filters\.CsrfPrevention
Filter\$LruCache(?:\$1)?|java\.util\.(?:Linked)?HashMap"/>
</Context>
2.配置管理用户并绑定角色
vim conf/tomcat-users.xml
<!--At:2020-4-6   By:zhouxy -->
  <role rolename="manager-gui"/>
  <user username="zhouxy" password="zhouxy" roles="manager-gui"/>
3.重启tomcat
sh bin/shutdown.sh
sh bin/startup.sh
4.访问控制台
192.168.1.1:8080/manager			输入用户名和密码即可


#四种角色
manager-gui:	允许访问html接口			  manager/html/*
manager-script： 允许访问纯文本接口		 manager/text/*
manager-jmx：允许访问jmx代理接口				manager/jmxproxy/*
manager-status：允许访问只读状态页面		 manager/status/*

注：这四种角色的定义是在webapps/manager/WEB-INF/web.xml中
```



```bash
#修改配置文件
vim /app/tomcat/conf/tomcat-users.xm																						
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users xmlns="http://tomcat.apache.org/xml"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
              version="1.0">
  <role rolename="admin-gui"/>
  <role rolename="host-gui"/>
  <role rolename="manager-gui"/>																											//定义仨 角色
  <user username="zhouxy" password="123" roles="admin-gui,host-gui,manager-gui"/>			//定义用户
</tomcat-users>
#重启
/app/tomcat/bin/shutdown.sh 				//关闭java后一定要检查进程和端口是否没了，否则启动时会有大量报错
ps -ef |grep java
root      17371   7458  0 20:30 pts/0    00:00:00 grep --color=auto java
ss -lntup |grep java
/app/tomcat/bin/startup.sh

#登陆
http://10.0.0.7:8080
用户名：zhouxy  密码：123
```

##### 部署jpress站点

```bash
#准备数据库
yum install -y mariadb-server
systemctl start mariadb
mysql
create database wordpress;
grant all on wordpress.* to jpress@'172.16.1.%' identified by '123';
grant all on wordpress.* to jpress@localhost indentified by '123';
mysql -h 172.16.1.51  -u jpress -p									//web01上测试
#部署代码
上传jpress-web-newest.zip
mkdir jpress && unzip jpress-web-newest.zip -d /app/tomcat/webapps
#登陆
http://10.0.0.7:8080/jpress/
输入：数据库wordpress 用户名jpress 密码123 数据库主机172.16.1.51
重启java即可

#写文章测试
登陆：10.0.0.7:8080/admin 写一篇文章，上传一张图片
文章在：MariaDB [wordpress]> select * from jpress_content limit 1 \G
图片在：ll /app/tomcat/webapps/jpress/attachment/20200319
```

##### 多实例

```bash
tar xf apache-tomcat-8.0.27.tar.gz
cp -a /apache-tomcat-8.0.27 /app/tomcat_8081
cp -a /apache-tomcat-8.0.27 /app/tomcat_8082
sed -i 's#8080#8081#g' www_tomcat8081/conf/server.xml
sed -i 's#8005#8006#g' www_tomcat8081/conf/server.xml
sed -i 's#8009#8010#g' www_tomcat8081/conf/server.xml
sed -i 's#8080#8082#g' www_tomcat8082/conf/server.xml
sed -i 's#8005#8006#g' www_tomcat8082/conf/server.xml
sed -i 's#8009#8010#g' www_tomcat8082/conf/server.xml

#启动多实例
/app/tomcat_8081/bin/startup.sh
/app/tomcat_8082/bin/startup.sh
#管理多实例
kill pid														//不能使用pkill
ps -ef |grep /tomcat/								//在启动、关闭、监控时一定要精确过滤
```

##### tomcat监控

```sh
当系统负载高，发现tomcat占用cpu很高时

1.精确定位是哪个tomcat进程导致的
jps -lvm	配置top命令								-l输出jar路径  -v输出jvm参数 -m输出method参数

2.导出进程详细信息及catalina.out日志
sudo -u tomcat /usr/java/latest/bin/jstack  25008 >/tmp/25008.jstack

3.导出jvm信息（堆栈内存和使用率等），用mat工具分析
/usr/java/latest/bin/jmap -heap 

/etc/init.d/nginx  stop
ps -ef |grep java
kill -9 25008
/etc/init.d/tomcat start
/etc/init.d/nginx  start


curl localhost:8080/cms/health.jsp -v
curl localhost/cms/health.jsp -v
```

##### tomcat安全优化

```bash
1.telnet管理端口保护：默认8005，改在8000~8999之间，把SHUTDOWN改为其它暗号
2.禁用管理端：默认8009，改在8000~8999间；通过iptables限制仅线上机器可访问ajp端口，防止线下测试流量被mod_jk上传
3.除权启动（监牢模式）
4.文件列表访问控制
5.隐藏版本信息
6.ajp连接端口保护
7.sever header重写
8.启停脚本权限回收
9.访问日志格式规范
10.访问限制

1. 配置部分（${ CATALINA_HOME }conf/server.xml）
<Server port="8527" shutdown=" dangerous">
<!-- Define a non-SSL HTTP/1.1 Connector on port 8080 -->
<Connector port="8080" server="webserver"/> 
<!-- Define an AJP 1.3 Connector on port 8528 -->
<!--Define an accesslog --> 
<Valve className="org.apache.catalina.valves.AccessLogValve"
directory="logs" prefix="localhost_access_log." suffix=".txt"
pattern="%h %l %u %t %r %s %b %{Referer}i %{User-Agent}i %D"
resolveHosts="false"/>
<Connector port="8528" protocol="AJP/1.3" />
<Context path="" docBase="/home/work/local/tomcat_webapps" debug="0" reloadable="false" 
crossContext="true"/>
2. 配置部分（${ CATALINA_HOME }conf/web.xml 或者 WEB-INF/web.xml）
<init-param>
<param-name>listings</param-name>
<param-value>false</param-value>
</init-param>
<error-page>
<error-code>403</error-code>
<location>/forbidden.jsp</location>
</error-page>
<error-page>
<error-code>404</error-code>
<location>/notfound.jsp</location>
</error-page>
<error-page>
<error-code>500</error-code>
<location>/systembusy.jsp</location>
</error-page>
3. 删除如下 tomcat 的默认目录和默认文件
tomcat/webapps/*
tomcat/conf/tomcat-user.xml
4. 去除其他用户对 tomcat 起停脚本的执行权限
chmod 744 –R tomcat/bin/*
```

```bash
ps -ef |grep java
/usr/java/latest/bin/jmap -heap 25008
 sudo -u tomcat /usr/java/latest/bin/jstack  25008 >/tmp/1.txt

```

##### 操作入门

```bash
#登陆
http://47.103.2.52			账号密码：admin admin


xceewzkypmgubhcj
```































