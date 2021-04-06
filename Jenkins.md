## 一、Jenkins安装

##### 传统安装

```bash
#下载安装配置启动
yum remove -y *jdk*
rpm -ivh https://repo.huaweicloud.com/jenkins/redhat/jenkins-2.256-1.1.noarch.rpm
yum install -y java-11-openjdk-devel
java -version
sed -i '/^JENKINS_USER/cJENKINS_USER="root"' /etc/sysconfig/jenkins 
systemctl start jenkins

#web及下载插件提速
sed -i 's#updates.jenkins.io#mirrors.tuna.tsinghua.edu.cn/jenkins/updates#g' /var/lib/jenkins/hudson.model.UpdateCenter.xml
sed -i 's#www.google.com#www.baidu.com#g' /var/lib/jenkins/updates/default.json
sed -i 's#updates.jenkins-ci.org/download#mirrors.tuna.tsinghua.edu.cn/jenkins#g' /var/lib/jenkins/updates/default.json 
systemctl restart jenkins

#登陆
http://192.168.50.128:8080	用户：admin	密码：cat /var/lib/jenkins/secrets/initialAdminPassword
```

##### docker安装

```bash
#起容器
docker pull jenkins/jenkins:lts
docker run --name jenkins -u root --restart unless-stopped -d -m 2G --memory-swap -1 -p 8080:8080 -p 50000:50000 -v /var/jenkins_home:/var/jenkins_home  --add-host git.hailian.com:192.168.2.1 reg.hailian.com/devops/jenkins:v1

镜像说明：
1. jenkins官方镜像停更2年，建议使用社区维护的jenkins/jenkins镜像，jenkins/jenkins:lts的jenkins版本是2.263.4，latest的是2.282

2.该镜像默认以jenkins用户(uid:1000)运行，容器内用户是jenkins，目录权限也是jenkins，这样的话无法直接创建持久卷，需要在linux上新建jenkins用户且uid为1000，再创建持久化目录并授权给jenkins用户。改用以root身份运行jenkins容器，会自动在宿主机上创建持久化目录/var/jenkins_home

3.-m是限制jenkins容器使用的内存，--memory-swap -1是不使用swap
4. 修改后的镜像地址是http://宿主机ip:8080，用户admin，密码hailian@2021
5.lts和latest镜像默认集成是jdk是1.8.0_282

#完成配置后保存镜像到harbor
docker commit -m 'install tools' -a 'zhouxy' ebe0b9b68269 reg.hailian.com/devops/jenkins:v1
docker push reg.hailian.com/devops/jenkins:v1
```

## 二、Jenkins配置

##### 安装常用插件

```bash
Localization: Chinese (Simplified)
Pipeline
Timestamper
AnsiColor
Git
DingTalk
```

##### 集成构建工具

```bash
由于jenkins从gitlab拉取代码后需要在本地对代码进行编译，所以需要在jenkins上安装maven、jdk、nodejs等

#在jenkins服务器上安装java、maven、nodejs，然后配置环境变量
wget http://mirrors.hust.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
wget https://cdn.npm.taobao.org/dist/node/v14.16.0/node-v14.16.0-linux-x64.tar.xz
tar xf apache-maven-3.6.3-bin.tar.gz -C /usr/lib
tar xf node-v14.16.0-linux-x64.tar.xz -C /usr/lib
mv /usr/lib/node-v14.16.0-linux-x64 /usr/lib/nodejs
mv /usr/lib/apache-maven-3.6.3 /usr/lib/maven

cat>> /etc/profile<<"EOF"
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export MAVEN_HOME=/usr/lib/maven
export NODE_HOME=/usr/lib/nodejs
export PATH=$PATH:${MAVEN_HOME}/bin:$JAVA_HOME/bin:$NODE_HOME/bin
EOF
source /etc/profile
java -version
mvn -v
node -v

#配置jenkens识别maven和jdk
管理jenkins - 全局工具配置 - Maven和JDK - name:maven3.6.3;MAVEN_HOME:/usr/local/maven/;去掉自动安装 -保存
管理jenkins - 系统配置 - 全局属性 - 环境变量 - 添加如下3个键值对- 应用并保存
		JAVA_HOME 	/usr/lib/jvm/java-11-openjdk
		MAVEN_HOME  /usr/lib/maven
		PATH+EXTRA  $M2_HOME/bin

#修改maven仓库为阿里maven仓库，本地maven仓库默认在~/.m2/repository/，可不用改
mvn help:effective-settings								#查看当前生效的配置
mvn help:active-profiles									#查看当前激活的配置
vim /usr/lib/maven/conf/settings.xml			#配置制品库为阿里的，注意：mirrorOf用于指定中央仓库，要用central
<settings>
  <mirrors>
    <mirror>
      <id>mirrorId</id>
      <name>Aliyun Maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
    </mirror>
  </mirrors>
</settings>

或者其它的
<mirrors>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云公共仓库</name>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云谷歌仓库</name>
        <url>https://maven.aliyun.com/repository/google</url>
    </mirror>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云阿帕奇仓库</name>
        <url>https://maven.aliyun.com/repository/apache-snapshots</url>
    </mirror>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云spring仓库</name>
        <url>https://maven.aliyun.com/repository/spring</url>
    </mirror>
    <mirror>
        <id>aliyunmaven</id>
        <mirrorOf>*</mirrorOf>
        <name>阿里云spring插件仓库</name>
        <url>https://maven.aliyun.com/repository/spring-plugin</url>
    </mirror>
  </mirrors>

```

##### 搭建Nexus私服

```bash
自动构建依然要通过maven打包，需要从国外官网下载大量的依赖，非常费时，可以搭建本地的maven仓库指向阿里云maven，再让jenkins打包时用本地maven仓库即可。本地仓库指定阿里云，下载依赖后积累到自己仓库

#部署私服
rpm -ivh jdk-8u181-linux-x64.rpm 
tar xf nexus-3.13.0-01-unix.tar.gz
mv nexus-3.13.0-01 /usr/local/nexus
[root@db01 /usr/local/nexus/bin]# ./nexus start					//启动nexus
打开页面：http://10.0.0.51:8081/		默认账号：admin 密码：admin123

私服添加阿里云maven仓库
齿轮-Repository-Reposiories-maven-central-Proxy(remote storage:https://maven.aliyun.com/nuxus/content/groups/public)

#让jenkins用私服
vim /usr/local/maven/conf/settings.xml
改此配置文件前先备份，具体修改见nexus.txt
```

##### 角色和凭据管理

```bash
#角色权限管理
可以为不同的用户分配不同权限，全局权限有：管理、读取、构建、配置、连接、新建、删除等等
工程角色：不同用户对不同项目有不同权限，如新建、取消、配置、构建、删除、查看工作区等，使用Pattern正则来为角色匹配项目如：longchuang.*表示longchuang开头的项目，其中.*里的.是必须要的

1.安装插件: Role-based Authorization Strategy
2.启动功能: 管理jenkins - 全局安全配置 - 授权策略 - 勾选Role-Based Strategy - 保存
3.先新建角色: 管理jenkins - Manage and Assign Roles - 管理角色 - 新建全局角色、项目角色、节点角色
4.再新建用户并分配角色: 管理jenkins - manage and assign roles - 分配角色

#凭据管理
凭据可用于存储需要密文保护的数据库密码、gitlab密码、harbor密码等，方便jenkins跟这些应用交互，凭据的类型有：用户名密码、gitlab API token、ssh用户及密钥、密码文件、加密字符串、通过上传证书等

安装插件：Credentials Binding Plugin
配置凭据：项目详情 - 凭据 - Folder - 全局凭据 - 添加凭据 

示例：jenkins通过ssh方式从gitlab上拉取代码
1、在任意机器生成密钥对，最好用jenkins服务器上的密钥
2、把公钥添加到gitlab: 用户 - 设置 - ssh密钥 - 复制公钥内容过来 - 添加密钥
3、把私钥添加到jenkins: 项目详情 - 凭据 - 添加凭据 - 类型选SSH - 用户名：root - Enter directly - 复制私钥
```

## 三、触发器

##### 触发器配置

```bash
Jenkins默认的触发器有4种：触发远程构建、其他工程构建后触发、定时构建、轮询SCM，安装插件Gitlab Hook Plugin和Gitlab Plugin就能用hook触发

#webhook触发
配置webhook触发需要安装Gitlab Hook Plugin和Gitlab Plugin两个插件，其原理是：一旦gitlab里代码有变化，就向jenkins的hook地址发传送代码的请求，jenkins收到代码自动完成构建,配置方法如下：

1.jenkins配置触发器：项目详情 - 构建触发器 - 选build when ** - 记录URL - 高级(指定分支) - Generate生成token 
2.jenkins上关闭终端验证：管理jenkins - 系统配置 - Gitlab - 去掉勾选的 Enable authe** end-point - 保存
3.gitlab开启hook功能：管理员登陆 - 管理中心 - 设置 - 网络 - 发外请求 - 勾选 Allow * from web hooks
4.gitlab里项目配置hook：项目详情 - 设置 - Integrations（集成） - 填写Gitlab webhook URL 其他默认 - 点Add

最后测试：点Test-选push events，出现 “Hook ececuted successfully:HTTP 200”就是能连通的

#其它工程构建后触发：当一个工程构建结束后自动触发本工程
配置：项目详情 - 构建触发器 - 选择build after other* - 关注的项目 - prejob - 保存
触发：在prejob详情页点立即构建，自动触发关联的项目


#不常用的触发配置
1.触发远程构建：在jengens上生成一个令牌，当用户通过浏览器访问令牌的url时就触发构建
配置：项目详情 - 构建触发器 （选择触发远程构建 - 输入身份验证令牌：加密后的字符串 - 得到URL ） - 保存
触发：浏览器输入 http://192.168.50.128:8080/job/longchuang-pipe/build?token=jiamizifuchuan

2.定时触发构建
配置：项目详情 - 构建触发器 - 选build periodically - 日程表输入： H/2 * * * * -保存
日程表里的-表示范围 /表示间隔多久执行一次  ,表示枚举 H是形参，随机值，为防止同时构建多个项目导致负载过高
H/30 * * * * 					表每30分构建一次
H H/2 * * * 					表每2小时构建一次
0 8,12,22 * *					表每天8点、12点、22点构建一次
H(0-29)/10 * * * *		表每小时的前半小时内每10分钟构建一次
H H(9-16)/2 * * 1-5		表工作日里9点到16点期间每2小时构建一次

3.轮询 Pol SCM
定时扫描本项目在gitlab中所有代码，如果代码有变化就触发，系统资源开销大，不推荐
```

##### 触发器pipeline

```groovy
在多分支流水线项目配置里触发器配置只有定时触发，不符合生产要求，所以需要在Jenkinsfile里写触发器

1.安装插件Gitlab Plugin
2.配置webhook，注意：需要手动生成随时字符串作为token，URL是本项目在jenkins上的地址
3.jenkensfile里添加配置，如下

def project_token = 'dsfjosidfsaf223r0dsf'
properties([
    gitLabConnection('gitlab-to-jenkins'),
    pipelineTriggers([
        [
            $class: 'GitLabPushTrigger',
            triggerOnPush: true,
            triggerOnMergeRequest: false,
            triggerOpenMergeRequestOnPush: "never",
            triggerOnNoteRequest: true,
            noteRegex: "Jenkins please retry a build",
            skipWorkInProgressMergeRequest: true,
            secretToken: project_token,
            ciSkip: false,
            setBuildDescription: true,
            addNoteOnMergeRequest: true,
            addCiMessage: true,
            addVoteOnMergeRequest: true,
            acceptMergeRequestOnSuccess: false,
            branchFilterType: "NameBasedFilter",
            includeBranchesSpec: "develop,release",
            excludeBranchesSpec: "",
        ]
    ])
])
```

## 四、代码审查

##### SonarQube安装

```bash
#软件安装
1.安装mysql并建sonar库
rpm -vih http://repo.mysql.com/mysql57-community-release-el7.rpm
yum install -y mysql-community-server
systemctl start mysqld
grep password /var/log/mysqld.log
mysql -p
> set password=PASSWORD("***");
> create database sonar;

#安装配置sonar
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-6.7.7.zip
unzip sonarqube-6.7.7.zip
mv sonarqube-6.7.7 /usr/lib/sonar
vim /usr/lib/sonar/conf/sonar.properties
16行：sonar.jdbc.username=root
17行：sonar.jdbc.password=X!@oy0ng
26行：取消注释
useradd sonar
chown -R sonar.sonar /usr/lib/sonar
su sonar /usr/lib/sonar/bin/linux-x86-64/sonar.sh start
tailf /usr/lib/sonar/logs/sonar.log 

#登陆SonarQube服务器
http://192.168.50.128:9000/				用户：admin  密码：admin

#生成token备用
zhouxy: c47f2b2359c221ed658ab2bf958df962cd2ccc52
```

##### 使用sonarQube

```bash
由SonarQube Scanner扫描代码，把结果上传到SonarQube服务器上做展示

#配置Jenkins连接Sonar服务器
1.安装SonarQube
插件：SonarQube Scanner
配置：管理jenkins - 全局工具配置 - 新增SonarQube Scanner - name: sonar-scanner - 版本：选最新 - 应用保存
2.添加凭据：jenkins管理 - 凭据 - 全局域 - 添加凭据 - 类型：secret text - 填入token - 名称：sonar-auth - 确定
3.添加SonarQube server：jenkins管理 - 配置 - Add SonarQube - name: sonar-server - URL：http://192..


#流水线项目代码审查
在项目根目录下新建文件sonar-project.properties，文件名是固定的，内容如下
sonar.projectKey=fly											#审查的项目
sonar.projectName=fly											#审查的项目名称
sonar.projectVersion=1.0						
sonar.source=.														#审查的目录 .代表所有目录
sonar.exclusions=**/test/**,**/target**		#排除哪些目录不审查
sonar.java.source=1.11
sonar.java.target=1.11
sonar.sourceEncoding=UTF-8

#在Jenkinsfile里添加代码审查，添加在拉取代码之后，构建之前
		stage("ScanCode"){
		    steps{
		        script {
		            scannerHome = tool 'sonar-scanner'
		        }
		        withSonarQubeEnv('sonar-server'){
		            sh "${scannerHome}/bin/sonar-scanner"
		     		}
		    }
		}
```

## 五、项目构建

##### 自由风格项目构建

```bash
在Jenkins新建项目后，配置源码及凭证指定从哪拉取代码，配置构建动作指定用mvn打包生成war包，配置构建后操作来把war包部署到web服务器。配置完成后点立即构建，此时jenkins先从gitlab上拉取代码到本地/var/lib/jenkins/workspace目录，然后执行mvn命令，最后部署到web服务器

#构建自由风格任务
创建：新建item - 输入任务名称 - 选择自由风格任务 - 确定 - 进入项目配置页面
配置：源码管理 - 选择Git - URL输入gitlab中该项目用ssh克隆的地址 - 凭据选有私钥的凭据
		 构建 - 增加构建步骤 - 选择Execute shell - 输入 mvn clean package - 保存
		 构建后操作 - 选择Deploy war/ear to a coontainer - 配置如下
		 		WAR/EAR files: 输入 target/*.war
				Context path: 不填
				Containers:	选择Tomcat 9.x Remote - 添加凭证（用户名：tomcat 密码：tomcat ；tomcat地址：http://192.168.50.100:8080
				Deploy on failure: 不选
构建：项目详情 - 立即构建

#maven项目构建
构建maven项目是专门为maven项目用的，需要安装插件：Maven Integration，其配置方法如下：
项目详情 - 构建（ Root POM：pom.xml ; Goals and options: clean package ） - 构建后操作（跟自由风格一致）
```

##### 流水线项目构建

```bash
需要安装插件：Pipeline，pipeline是一套工作流框架，把原本独立的任务连接起来，以实现单个任务难以完成的复杂流程编排和可视化工作。pipeline分为声明式和脚本式，两者功能一致，声明式以pipeline{}开头，脚本式以node{}开头，jenkins2.0后，官方推荐用声明式。可能通过pipeline语法工具来生成复杂的语句，把生成的pipeline复制到gitlab根目录并命名为Jenkinsfile

#构建Pipeline项目
创建：新建item - 输入任务名称 - 选择流水线 - 确定 - 自动进入项目配置页面
配置：构建触发器见第3章第5节-定义-选择pipeline script from SCM - SCM选择git - 再填写git地址 - 选择凭证 - 保存
```

##### 多分支流水线构建

```bash
创建：	新项目 - 输入项目名 - 选择多分支流水线 - ok
配置： 选择项目 - 配置- 分支来源 - git - save (保存后该项目下自动生成各分支)
			1.需要先在gitlab先生成token:	用户 - 设置 - 访问令牌 - 输入令牌名并勾选api - 生成 - 记录下token
			2.再回到jenkins填写git仓库配置
				Project Repository:git@192.168.0.146:zhouxy/energyconsumption.git
 				Credentials	: ADD - longchuang* - kind选GitLab API token - 填写在gitlab生成的token
 				Behaviours:	Discover branches
 				Property strategy:	All branches get the same properties
```

##### 参数化构建

```bash
在jenkins的项目里配置参数化构建后，在jenkins项目面板上立即构建变成了build with parameters，可选择不同参数来构建

1.jenkins项目开启参数化构建： 项目详情 - General - This project is parameterized - 添加参数 - 名称是变量名
	选项：master develop release 每行一个，第一行是默认选项
2.修改gitlab上项目里的Jenkinfile，把文件里所有master替换为${branch} - 提交到gitlab
3.jenkins上构建：项目 - build with parameters - 选择参数 - 开始构建
```

##### 日志文字加颜色

```bash
使用插件：AnsiColor，引用是需要在options添加，使用时直接用

options{ansiColor('xterm')}
echo "\033[31m redis中用户数据不相等! 预设值为${users_preset}，而redis中存在件数是${users_inredis} \033[0m "}

#其它选项
echo  "\033[41;30m红底黑字\033[0m"
echo  "\033[30m 黑色字 \033[0m"
echo  "\033[31m 红色字 \033[0m"
echo  "\033[32m 绿色字 \033[0m"
echo  "\033[33m 黄色字 \033[0m"
echo  "\033[46;30m 天蓝底黑字 \033[0m"
echo  "\033[4;31m 下划线红字 \033[0m"
echo  "\033[5;34m 红字在闪烁 \033[0m"
```

## 六、项目部署

##### 发送远程命令部署

```bash
在Jenkins服务器上发送远程命令给生产环境，需要安装插件Publish Over SSH

#配置jenkins连接远程服务器

1. jenkins上的公钥推送到远程服务器
ssh-copy-id -i ~/.ssh/id_rsa.pub 10.1.20.125

2.jenkins上配置远程连接
管理jenkins - 系统配置 - Publish over ssh - path to key填本机私钥地址 - 新增 - username是本机生成密钥的用户 remote directory用/

3.用流水线语法生成远程调用命令 sshPublisher

sshPublisher(publishers: [sshPublisherDesc(configName: 'metro-server6', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: 'll', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
```

##### 部署到k8s集群

```bash
jenkins通过把master节点上的./kube/config的配置存入凭据里，用凭据ID来连接k8s集群
需要的插件：Kubernetes Continuous Deploy Plugin

准备：在k8s集群的master节点执行命令 cat ~/.kube/config 复制所有内容
配置：添加全局凭据 - 类型选Kubernetes * -描述 k8s-uat-config - kubeconfig选直接输入 - 粘贴复制的内容
使用：写在Jenkinsfile里的配置如下
stage('按分支部署工程') {
    steps {
        script{
            sh """
                sed -i "s#IMAGE_NAME#$image_url:$image_tag#g" deploy.yml
                sed -i "s#PROJECT_NAME#$project_name#g" deploy.yml
            """
            if (env.BRANCH_NAME == 'develop') {
                sh "kubectl --kubeconfig=/root/.kube/config-herelink-develop apply -f deploy.yml --record"
            } else if (env.BRANCH_NAME == 'release') {
                sh "kubectl --kubeconfig=/root/.kube/config-herelink-release apply -f deploy.yml --record"
            } else (env.BRANCH_NAME == 'master') {
                sh "kubectl --kubeconfig=/root/.kube/config-herelink-master apply -f deploy.yml --record"
            }
        }
    }
}
```

## 七、结果通知

##### 通知到Gitlab

```bash
在Jenkins里配置把执行结果反馈给gitlab，此时在gitlab项目面板上项目后有个绿色圈内有对号代表构建成功，红色圈内有x代表构建失败

1.先在gitlab上生成token
	用户 - 设置 - 访问令牌 - 填写名称gitlab-api-token - 勾选api - 生成token - 记录token（仅生成时显示1次）

2.再到Jenkins添加token 
	管理 - 系统设置 - Gitlab - 连接名：gitlab - url填http://192.168.0.146 - 凭据用gitlab api token - 测试
3.Jenkins发送状态
	项目详情 - 配置 - 构建后操作 - 选择publish build status to Gitlab commit - 保存
```

##### 通知到邮件

```bash
当jenkins构建成功后，发送邮件，可用Email Extension Template插件来定制邮件模板
提前条件：邮箱要开启stmp功能

配置：管理jenkins - 系统配置 - Extended E-mail Notification - 配置如下
				Jenkins Location - 系统管理员邮件地址: abc@qq.com		#填写发件人的地址
				SMTP Server: smtp.exmail.qq.com										#发送邮件的服务器
				SMTP Port: 465																		#开启ssl就是465 不开启ssl默认25
				SMTP Username: zhouxiaoyong@longchuang.com				#发件邮箱
				SMTP Password: ***																#密码是授权码
				Use SSL: on																				#开启ssl
				Default user E-mail suffix: longchuang.com				#邮箱后缀
				Default Content Type: HTML(text/html)							#不使用纯文本发邮件
未完成，待添加
```

##### 通知到钉钉

```bash

```

## 八、Pipeline

##### 语法规则

```groovy
1.所有的声明式Pipeline都必须包含一个pipeline块中
2.流水线顶层必须是一个块，特别是pipeline{}
3.不需要分号作为分割符，是按照行分割的
4.语句块只能由阶段、指令、步骤、赋值语句组成。例如: input被视为input()
5.脚本式Pipeline： 不需要step关键字
```

##### 基本框架

```groovy
#!groovy												//幻数，用于指定代码的解释器
@Library('jenkinslib') _				//导入共享库，_是让生效的意思
String workspace = "xxx"				//定义变量

pipeline{
	agent{}												//用于指定pipeline在哪个jenkins节点上执行
	options{}											//用于配置选项，如时间戳、超时时间、可否并行等
	environment{}									//用于定义环境变量，可放在stage层表对本stage生效
	paramters{}										//用于定义参数，如字符型、布尔型、
	trigger{}											//用于定义触发器
	tools{}												//用于获取在全局工具配置中定义的环境变量
	input{}												//用于在执行各阶段时由人工确认是否继续
	stages{												
		stage("并行修改代码"){				//并行项目只能有一个parallel阶段，且该阶段不可再做嵌套
			when{branch master}
			failFast true							//当本阶段任一进程失败时，强制停止所有并行项目
			parallel{
				stage("job1"){steps{}}
				stage("job2"){steps{}}
			}
		}
		stage("项目构建"){
			steps{
				echo "Hello world"
				script{ println("good")
			}
		}
		stage("项目部署"){}
	}
	post{													 //跟stages平级，用于指定构建后操作
		always{echo '无论流水线或阶段的完成状态如何，都允许在 post 部分运行该步骤'}
    changed{echo '只有当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 post 部分运行该步骤'}
    failure{echo '只有当前流水线或阶段的完成状态为"failure"， 通常web UI是红色'}
    success{echo '只有当前流水线或阶段的完成状态为"success"， 通常web UI是蓝色或绿色'}
    unstable{echo '只有当前流水线状态为"unstable"，通常由于测试失败,代码违规等造成。通常web UI是黄色'}
    aborted{echo '只有当前流水线或阶段的完成状态为"aborted"，通常由于流水线被手动的aborted。通常web UI是灰色'}
	}												
}
```

##### 共享库

```bash
在Gitlab代码目录定义类：/root/src/org/devops/tools.grooy
package org.devops

def PrintMsg(value,color){
    colors = ['red'   : "\033[40;31m >>>>>>>>>>>${value}<<<<<<<<<<< \033[0m",
              'blue'  : "\033[47;34m ${value} \033[0m",
              'green' : "[1;32m>>>>>>>>>>${value}>>>>>>>>>>[m",
              'green1' : "\033[40;32m >>>>>>>>>>>${value}<<<<<<<<<<< \033[0m" ]
    ansiColor('xterm') {
        println(colors[color])
    }
}

在Jenkins定义库：管理中心 - 系统配置 - 全局流水线库 - 自定名称：jenkinslib - 默认版本：master - 指定git仓库 - 保存

在Jenkinsfile里引用：
	@Library('jenkinslib') _
	def tools = new org.devops.tools()
	steps{script{tools.PrintMsg("打印的文字"),"green"}}
```

##### 环境变量

```bash
#定义与调用
environment{
	name = "zhouxy"
}
steps{ 
    sh 'printenv'									//打印出所有环境变量
		echo "${name}"						
		echo "${env.JENKINS_URL}"
}

#def定义变量（必须放在pipeline前面或者script语句块里面）
def projectName = "longchuang-software"
def podName=sh(script:"kubectl get pod|grep publish|awk '{print\$1}'",returnStdout:true).trim()

#多分支流水线内置环境变量
BRANCH_NAME									当前分支名称
CHANGE_ID										用于标明变更ID，但Pull Request	不支持的情况下此环境变量会被unset
CHANGE_URL									用于标明变更的URL	不支持的情况下此环境变量会被unset
CHANGE_TITLE								用于标明变更的标题	不支持的情况下此环境变量会被unset
CHANGE_AUTHOR								用于标明提交变更的人员的名称	不支持的情况下此环境变量会被unset
CHANGE_AUTHOR_DISPLAY_NAME	用于标明提交变更的人员的显示名称	不支持的情况下此环境变量会被unset
CHANGE_AUTHOR_EMAIL					用于标明提交变更的人员的邮件地址	不支持的情况下此环境变量会被unset
CHANGE_TARGET								用于合并后的分支信息等	不支持的情况下此环境变量会被unset
#公共变量
BUILD_NUMBER								当前的构建编号
BUILD_ID										表示当前构建ID	使用YYYY-MM-DD_hh-mm-ss的时间戳以表示之前的构建信息
BUILD_DISPLAY_NAME					当前构建的显示信息
JOB_NAME										构建Job的全称，包含项目信息
JOB_BASE_NAME								除去项目信息的Job名称
BUILD_TAG										构建标签,生成的形为jenkins-JOB_NAME的构建标签
EXECUTOR_NUMBER							执行器编号，用于标识构建器的不同编号。	编号从0开始
NODE_NAME										构建节点的名称	如果在master节点上执行的话，名称为master
NODE_LABELS									节点标签
WORKSPACE										构建时使用的工作空间的绝对路径
JENKINS_HOME								JENKINS根目录的绝对路径	用于指定Jenkins的Master节点数据存储的路径
JENKINS_URL									Jenkins的URL信息	只有当系统配置中被设定才会显示。
BUILD_URL										构建的URL信息	只有当系统配置中被设定才会显示。
JOB_URL											构建Job的URL信息	只有当系统配置中被设定才会显示。
GIT_COMMIT									git提交的hash码
GIT_PREVIOUS_COMMIT					当前分支上次提交的hash码	仅在信息存在的情况下才会显示
GIT_PREVIOUS_SUCCESSFUL_COMMIT	当前分支上次成功构建时提交的hash码	仅在信息存在的情况下才会显示
GIT_BRANCH									远程分支名称，仅在信息存在的情况下才会显示
GIT_LOCAL_BRANCH						本地分支名称
GIT_URL											远程URL地址	当存在多个URL地址的情况下，引用方式依次为GIT_URL_1、 GIT_URL_2等
GIT_COMMITTER_NAME					Git提交者的名称
GIT_AUTHOR_NAME							Git Author的名称
GIT_COMMITTER_EMAIL					Git提交者的email地址
GIT_AUTHOR_EMAIL						Git Author的email地址
MERCURIAL_REVISION					Mercurial的版本ID信息
MERCURIAL_REVISION_SHORT		Mercurial的版本ID缩写
MERCURIAL_REVISION_NUMBER		Mercurial的版本号信息
MERCURIAL_REVISION_BRANCH		分支版本信息
MERCURIAL_REPOSITORY_URL		仓库URL信息
SVN_REVISION								Subversion的当前版本信息
SVN_URL											当前工作空间中被checkout的Subversion工程的URL地址信息	
示例：
13:51:39  + printenv
13:51:39  JENKINS_NODE_COOKIE=5d6bcdac-8b91-4674-9c5e-d663f1b9b074
13:51:39  BUILD_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/127/
13:51:39  SHELL=/bin/bash
13:51:39  HUDSON_SERVER_COOKIE=d91b3818746020a9
13:51:39  STAGE_NAME=检测外部配置是否改动
13:51:39  BUILD_TAG=jenkins-longchuang-publish-develop-127
13:51:39  BRANCH_NAME=develop
13:51:39  GIT_PREVIOUS_COMMIT=c6817a255166202654a1e29455818dd79cb330d5
13:51:39  JOB_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/
13:51:39  WORKSPACE=/var/lib/jenkins/workspace/longchuang-publish_develop
13:51:39  RUN_CHANGES_DISPLAY_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/127/display/redirect?page=changes
13:51:39  RUN_ARTIFACTS_DISPLAY_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/127/display/redirect?page=artifacts
13:51:39  GIT_COMMIT=b89a9bfb0648b34e5a5239c6646707fa0fd93c7c
13:51:39  JENKINS_HOME=/var/lib/jenkins
13:51:39  PATH=$MAVEN_HOME/bin:$MAVEN_HOME/bin:/sbin:/usr/sbin:/bin:/usr/bin
13:51:39  RUN_DISPLAY_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/127/display/redirect
13:51:39  _=/bin/printenv
13:51:39  PWD=/var/lib/jenkins/workspace/longchuang-publish_develop
13:51:39  JAVA_HOME=/usr/lib/jvm/java-11-openjdk
13:51:39  HUDSON_URL=http://192.168.50.128:8080/
13:51:39  LANG=zh_CN.UTF-8
13:51:39  JOB_NAME=longchuang-publish/develop
13:51:39  BUILD_DISPLAY_NAME=#127
13:51:39  BUILD_ID=127
13:51:39  JENKINS_URL=http://192.168.50.128:8080/
13:51:39  GIT_PREVIOUS_SUCCESSFUL_COMMIT=c6817a255166202654a1e29455818dd79cb330d5
13:51:39  JOB_BASE_NAME=develop
13:51:39  RUN_TESTS_DISPLAY_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/127/display/redirect?page=tests
13:51:39  M2_HOME=/usr/local/maven
13:51:39  HOME=/root
13:51:39  SHLVL=4
13:51:39  GIT_BRANCH=develop
13:51:39  WORKSPACE_TMP=/var/lib/jenkins/workspace/longchuang-publish_develop@tmp
13:51:39  EXECUTOR_NUMBER=2
13:51:39  JENKINS_SERVER_COOKIE=durable-db35de083c426d1d33be98c4a69d45d5
13:51:39  NODE_LABELS=master
13:51:39  GIT_URL=git@192.168.0.146:aiotv1.1/longchuang-cloud-base-platform-service/longchuang-publish.git
13:51:39  HUDSON_HOME=/var/lib/jenkins
13:51:39  NODE_NAME=master
13:51:39  BUILD_NUMBER=127
13:51:39  JOB_DISPLAY_URL=http://192.168.50.128:8080/job/longchuang-publish/job/develop/display/redirect
13:51:39  HUDSON_COOKIE=17844838-d27d-4184-bd3d-9c2bb15d716b
13:51:39  NODE_HOME=/usr/local/nodejs
```

## 九、Groovy语法

##### 安装解释器

```bash
#mac
brew install groovy
export GROOVY_HOME=/usr/local/opt/groovy/libexec
groovyconsole
groovysh
groovy foo.groovy
```

##### 字符处理

```bash
#字符处理
可用单引号、双引号、3个双引号、3个单引号来表示字符串，区别是单引号和3单引号表强引用，不解释变量，双引号会解释变量

"testmysqlself".contains("tes")						#判断是否包含，返回true或false
"testmysqlself".endsWith("tes")						#判断是否以tes结尾，返回true或false
"testmysqlself".size()										#计算字符串长度
"testmysqlself".length()									#同上
"testmysqlself"+"jonny"										#字符拼接，结果是testmysqlselfjonny
"testmysqlself"-"se"											#字符剪切，结果是testmysqllf
"name.substring(m,n)"											#截取字符串，截取从m号索引到n号索引这一段字条，索引是从0开始的
"testmysqlself".reverse()									#倒序
"testmySQLself".toUpperCase()							#小写的部分转大写
"testmySQLself".toLowerCase()							#大写的部分转小写
"host01,host02,host03".split(',')					#分割字符串为列表，结果是[host01, host02, host03]

用法示例：
	hosts = "host01,host02,host03".split(',')
	for (i in hosts)
	println(i)

#列表
[1,2,3,4]			
[11,2,3,5].sum()													#元素里总和
[11,2,3,5].max()													#最大值
[11,2,3,5].mix()													#最小值
[11,2,3,5].isEmpty()											#判断列表是否为空，空返回true
[11,2,3,5].contains(3)										#判断是否包含，返回true或false
[11,2,3,5].sort().reverse()								#sort是从小到大排序，reverse是倒序
[11,2,3,5].each{println it}								#遍历列表，遍历结果默认赋给变量it,结果是11 2 3 5
[1,2,3,4] + 5															#添加元素5，也可用[1,2,3,4]<<5
[11,2,3,5] -5															#减少元素5
[11,2,3,5].remove(0)											#去掉第0号元素
[1,1,1,2,3].unique().join("-")						#unique()是去重，join()是指定连接符
`[11,2,3,5].count()												#计算总个数

#map是kv结构的列表
[a:100,b:200,c:300]												#定义一个map
[a:100,b:200,c:300].isEmpty()							#判断是否为空
[a:100,b:200,c:300] + [d:400]							#添加元素
[a:100,b:200,c:300] - [c:300]							#去掉元素
[a:100,b:200,c:300].remove("a")						#去掉元素
[a:100,b:200,c:300].removeALL("a")
[a:100,b:200,c:300].keySet()							#生成key的列表
[a:100,b:200,c:300].values()							#生成value的列表
[a:100,b:200,c:300].size()								#计算元素总数
[a:100,b:200,c:300].each{print it}				#遍历元素
```

##### 内置命令

```bash
#目录和文件
steps{dir("/var/logs") { deleteDir() }}											#删除目录
steps{dir("common-core")sh{"${M2_HOME}/bin/mvn install"}}		#切换目录执行命令

#创建文件
script {
		writeFile(file: ./docker-config.yml text:"very good",encoding: "utf-8")
		def content = readFile(file: ./docker-config.yml,encoding:"UTF-8")
		echo "${content}"
}

#判断文件是否存在
steps{
  script{
		if (fileExists('/tmp/a.txt')){echo '有'}
    else {echo '没有'}
	}
}

#交互式确认
stage("交互式确认信息"){
    input{
        message "are you ok?"				 //对话框头部信息
        ok "确定"											//写在确定按钮上的信息
        parameters {								 //把输入的信息存到变量PERSON
            string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
        }
    }
    steps{echo "Hello, ${PERSON}, nice to meet you."}
} 

#根据执行结果返回不同状态码，如下过滤到path字符，则返回状态码0，过滤不出来则返回1
steps{script{
    result = sh (script : "tail /var/log/messages|grep path", returnStatus:true)		
    echo "result:$result"																#结果是$result=0
  }}

#抛出异常
steps{
    script{
    if (fileExists('/tmp/a.txt')){echo '文件存在'}
    else {error ("文件不存在")}
}}

#重复执行块
执行N次闭包内的脚本。如果其中某次执行抛出异常，则只中止本次执行，并不会中止整个`retry`的执行。同时，在执行`retry`的过程中，用户是无法中止pipeline的。

steps {
    retry(20) {
	    script {
		    sh script: 'curl http://example', returnStatus: true
        }
    }
}

#等待条件满足
steps{script{
  def podName=sh(script:"kubectl get pod|grep ${projectName}|egrep 'Running|ContainerCreating'|awk '{print\$1}'",returnStdout:true).trim()
  timeout(3){
    waitUntil(initialRecurrencePeriod: 3000) {
      def result = sh (script: "kubectl logs ${podName}| grep 'JVM running'",returnStatus:true)
      return (result == 0)
    }
  }  
}}


#制品：保存取用临时文件，用name来标识保存的临时文件合集，为字符串型
stash(name: "deployFile", includes: "longchuang*.yml",allowEmpty: true)
unstash("deployFile")
```

##### 判断

```groovy
//when用于stage内部
when {branch 'master'}
when {not {branch 'master'}}
when {expression {return params.DEBUG_BUILD}}																	//当表达式返回值为真时
when {allOf {branch 'dev'; environment name:'DEBUG_TO',value:'production'}}		//所有条件满足则为真
when {anyOf {branch 'dev' ; branch 'rel'}}																		//任一条件满足即为真

//判断变量是否存在
when {express {env.CHANGE_ID != null}}
if (env.postgresql != null) {echo  "本环境连接的PostgrepSQL是: $postgresql" }else { echo "本环境未连接postgresql"}


//判断文件是否存在
steps{
	script{
		if (fileExists('/tmp/a.txt')){echo '有'}
		else {error ("文件不存在，中止流水线")}
	}
}

//if判断
stage("检查服务是否正常启动"){
    steps{
        script{
            buildType = "maven"
            if (buildType == "maven"){ println("this is maven")}
            else if (buildType == "ant"){println("this is ant")}
            else {println("Type error")}
        }
    }
}

//if多分支
script{
    sh """
        sed -i "s#IMAGE_NAME#$image_url:$image_tag#g" deploy.yml
        sed -i "s#PROJECT_NAME#$project_name#g" deploy.yml
    """
    if (env.BRANCH_NAME == 'develop') {
        sh "kubectl --kubeconfig=/root/.kube/config-herelink-develop apply -f deploy.yml --record"
    } else if (env.BRANCH_NAME == 'release') {
        sh "kubectl --kubeconfig=/root/.kube/config-herelink-release apply -f deploy.yml --record"
    } else (env.BRANCH_NAME == 'master') {
        sh "kubectl --kubeconfig=/root/.kube/config-herelink-master apply -f deploy.yml --record"
    }
}

//swatch判断
stage("检查服务是否正常启动"){
    steps{
        script{
            buildType = "ant"
            switch("${buildType}"){
                case 'maven' :  println("this is maven");break  ;;   
                case 'ant'   :  println("this is ant");break    ;;
                default      :  println("Type error")           ;;
            }
        }
    }
}    

```

##### 循环

```groovy
//for循环
langs=['java','python','groovy']
for (lang in langs){
    println("$lang")
}

//while循环
while (3>2){
	println("right")
}
```

##### 函数

```groovy
//定义函数，调用传入的参数用${xx}
def CheckDatebase(database,table,num){
    def NUM=sh(script:"psql -Upostgres -h 10.1.20.82 -w -t -d $database -c 'select count(*) from $database.$table'",returnStdout:true).trim()
    if("$num" == "$NUM"){
        echo "\033[32m${database}库中${table}表实际件数与预期相等，预期：${num}，实际：${NUM}\033[0m"
    }else{
        echo "\033[31m${database}库中${table}表实际件数与预期不符，预期：${num}，实际：${NUM}\033[0m"
    }
}


//调用函数，参数需要用双引号
steps{
    CheckDatebase("adminauthority","admin_user","1")
    CheckDatebase("adminauthority","admin_authority","36")
    CheckDatebase("adminauthority","admin_role","1")
    CheckDatebase("adminauthority","admin_user_role","1")
    CheckDatebase("adminauthority","admin_role_authority","36")
    CheckDatebase("masterdata","area","3290")
    CheckDatebase("masterdata","data_dictionary_type","2")
}
```

##### 常用DSL

```bash
Domain Specific Language 领域特定语言
checkout: Check out from version control
publishHTML: Publish HTML reports
input: Wait for interactive input
buildUser			=> with
withCredentials: Bind credentials to variables
cleanWs: Delete workspace when build is done
```

##### 校验

```bash
#校验配置文件
stage('检测外部配置是否改动'){
  steps{
    sh "md5sum -c ../md5/${projectName}.md5"
    sh "md5sum ${WORKSPACE}/pom.xml ${WORKSPACE}/src/main/resources/bootstrap.yml  >../md5/${projectName}.md5"
	}
}

#判断文件是否存在
steps{
	script{
		if (fileExists('/tmp/a.txt')){echo '有'}
		else {error ("文件不存在，中止流水线")}
		else {echo '文件不存在'}
	}
}
```









