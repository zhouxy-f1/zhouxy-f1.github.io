## 一、安装部署

##### 安装配置启动

```bash
GitLib使用Git作为代码管理工具，集成了postgresql、redis、alertmanager、gitaly、logrotate、nginx、puma、prometheus、frafana等组件，在此基础搭建web服务，通过web访问项目，非常易于浏览提交过的版本并可做权限控制

#安装gitlab-ce-13.3.0
cat>/etc/yum.repos.d/gitlab-ce.repo<<"EOF"
[gitlab-ce]
name=Gitlab CE Repository
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el$releasever/
gpgcheck=0
enabled=1
EOF
yum makecache
yum install -y gitlab-ce

#配置（完成后自动启动）
sed -i "/^external_url/cexternal_url 'http://192.168.1.120'"  /etc/gitlab/gitlab.rb
echo "192.168.1.120 www.gitlab.com">>/etc/hosts
gitlab-ctl reconfigure

#登陆及简单优化 (先在自己电脑上配置hosts解析)
http://www.gitlab.com	账户：root 密码：首次登陆自定义，老版本默认是root12345

中文显示：	用户 - setting - Preferences - Localization-language - 简体中文 - Save changes
界面外观：	管理中心-外观-填写联系方式上传公司logo等-更新
去注册页：	设置-settings-sign up restrictions - sign up enabled 去掉勾-保存

#gitlab服务器管理维护
/opt/gitlab/											#gitlab的程序安装目录
/var/opt/gitlab										#gitlab目录数据目录 
/var/opt/gitlab/git‐dfata 				#存放仓库数据
gitlab-ctl reconfigure 						#更改配置文件后需重新配置   
gitlab-ctl status [nginx]					#查看目前gitlab所有服务运维状态 
gitlab-ctl stop	[nginx]						#停止gitlab所有，或单独停nginx
gitlab-ctl tail	[nginx]						#查看所有服务的日志，或单独nginx的
```

## 二、新建项目

##### 管理员新建项目

```bash
gitlab里项目管理是用群组来区分，一个群组里可以有多个项目，群组可以邀请新成员或其它群组的成员进本群组；每个项目也可以邀请新成员或另一个群组的成员加入本项目，成员的权限是通过其加入项目时的角色来限制。

#新建项目并邀请成员
新建用户：管理中心-用户-新用户-填写用户名等信息-创建用户-编辑-设置初始密码-保存		#用户首次登陆需要修改密码
新建群组：管理中心-群组-新建群组
新建项目：管理中心-项目-新建项目-在项目URL里选择归属的群组-新建
项目添加成员：项目-成员-邀请成员或群组-邀请
群组添加成员：群组-成员-邀请成员或群组-邀请
删除项目：管理中心-选择项目-详情-设置-通用-高级-删除项目
删除群组：管理中心-概览-群组-删除																		 #删除群组会删掉组内所有项目、合并请求等
删除用户：管理中心-用户-设置-删除用户																	#删除用户可选择性删除用户及其所有贡献

#上传代码
在上传代码之前需要先把本机的公钥添加到gitlab服务器
[zhouxy@Mac ~]% cat .ssh/id_rsa.pub 把结果复制到-用户-设置-ssh密钥-添加密钥

1.通过git命令上传本地代码
	cd longchuang
	git init
	git config --global user.name "Administrator"
	git config --global user.email "admin@example.com"
	git add .
	git commit -m "first commit"
	git remote add origin git@10.0.0.88:dev/longchuang.git
	git push -u origin --all

2.IDEA使用Git，在IDEA上操作如下
	检查git是否配置好: Preferences - Version Control - Git - Test			=> 弹出版本号则配置无误
	创建本地仓库: VCS - Import init Version Control - Create Git Repository - 选择目录
	首次提交：右击项目名 - Git - Add后再Commit directory - 选择要上传的文件写备注 - Commit
	上传代码：右击项目名 - Git - Repository - Remotes - 添加远程仓库 - Push
	注意：添加远程仓库前，先把当下运行IDEA的用户zhouxy的公钥添加到gitlab，否则提示输入密码

#分支保护
新建多分支后，默认保护master分支，只允许所有者合并或上传代码。如果需要保护其它分支不让合并或上传代码，操作如下：
项目详情-设置-仓库-保护分支-选择需要被保护的分支-选择权限 - 保护


#等待开发写完代码提交合并申请
项目详情-Merge Requests - merge
```

##### 程序员上传代码

```bash
#创建好成员就可以登陆开发人员自己的分支仓库
登陆：10.0.0.88 用户名：zhouxy 密码zhouxy12	登陆后强制先改密码
查得远程仓库地址: git@10.0.0.88:dev/longchuang.git

添加ssh keys
右上角头像-settings - SSH Keys- Add key

#从仓库拉取代码到本地,修改完再上传到dev分支
git clone git@10.0.0.88:dev/longchuang.git
cd longchuang/
vim index.html
git commit -am "modify index.html"
git branch dev
git checkout dev
git push -u origin dev

#开发申请把dev分支合并到master
项目-左边栏选Merge requests-new merge request-发起申请-填写标题 内容-submit serge request
运维在页面相同位置同意请求即可，还可以删除源分支
```

## 三、常用命令

##### 全局配置

```bash
cd code																			#进入code目录
git config --global user.name  "zhouxy"			#配置提交代码时的用户名
git config --global user.email "*@qq.com"		#配置提交代码时的邮箱
git config --global color.ui true						#配置颜色
git config -l																#查看当前git的配置信息
git config -e																#编辑git的配置文件
:> ~/.gitconfig															#清除配置
```

##### 提交代码

```bash
git add [file1] dir1				#把文件file1和目录dir1及其子目录添加到暂存区，进入暂存区代表开始追踪该文件和目录 
git add .										#把当前目录所有文件及目录加入暂存区
git status									#查看有变化的文件
git rm -f file1							#删除暂存区和工作区的file1
git rm --cached file1				#删除暂存区的file1
git	mv file1 file2					#文件改名，工作区和暂存都会改
git commit -m "message"			#把暂存区所有文件提交到本地仓库
git commit file1 -m "xx"		#把暂存区中file1提交到本地仓库
git commit -a 							#把工作区直接提交到本地仓库
git branch									#查看本地分支
git branch -r								#查看远程分支
git branch -a								#查看本地和远程的分支
git branch dev							#新建dev分支，但依然留在当前分支
git checkout -b ops					#新建ops分支，并切换到ops分支
git checkout dev						#切换到dev分支
git checkout -							#切换到上次所在的分支
git checkout -- a						#从暂存区恢复a,覆盖本地a
git diff										#比对工作目录与暂存区文件内容的不同处
git diff --cached						#比对暂存区与仓库文件内容
git commit -m "modifyd a "	#把暂存区的文件提交到仓库，以哈希算法存于./git/object里，无法撤回，除非删了object
git commit -am "xxx"				#不经暂存区，直接提交到仓库 
git log 										#查看历史操作
git log  --oneline					#只查历史中8位哈希和message
git log --decorate					#查看当前快照的指针，意思是当前在哪个快照下工作
git reflog									#查看所有历史提交记录
git reset --hard ********		#本地目录状态回到任一历史状态，可前滚可回滚
```

##### 代码合并

```bash
master分支：仅用于发布新版本，一定要保障主分支的稳定
其它分支：如果要开发新功能，先把主分支克隆到本地，修改完合并到主分支
git branch release					#切换到release分支
git merge develop						#在本地把develog分支合并到当前分支，当前分支为release
git push -u origin release	#上传release分支到gitlab
```

##### 标签及其它

```bash
#使用标签来设置版本号
git tag -a v1.0.1 -m "complicated"						#对当前代码打标签，-a 批定标签
git tag																				#查看标签
git tag -a v1.2.1 3702353 -m "history tag"  	#给历史提交的代码打标签
git reset --hard v1.1.1												#根据标签回滚到v1.1.1
git tag -d v1.2.1															#删除本地标签（仅标签）
git push origin :refs/tags/[tagName]					#删除远程标签	

#其它
git branch 									#查看本地分支
git branch -r								#查看远程分支
git branch -a								#查看本地和远程的分支
git remote -v								#查看远程仓库
git branch develop					#创建分支
git checkout develop				#切换当前到test分支
git checkout -							#切换到上一个分支
git checkout -b develop			#创建并切换分支
git branch -d develop				#合并完分支后立即删除当前分支，否则看不到master上其它人的代码，要重新创建拉取
git merge testing						#在主分支上合并testing分支，先用git branch先查当前分支为主分支
git checkout master
git merge dev
vim aa.txt 去掉特殊字符和不要的内容，最后提交就行了
git stash				保存现场
git stash pop		返回现场
```

## 四、Github

```bash
Github是全球最大的软件仓库，用于托管Git版本库，一旦托管代码就开源了谁都可以克隆，哪怕自己去删了都没用，买私有库$7/月
1.注册用户
2.配置ssh-key
3.创建项目
4.克隆项目到本地
5.推送新代码到github

git remote																								#查看远程仓库
git remote add origin git@github.com:zlwm1152/linux-.git	#连接远程仓库
git remote remove origin																	#删除origin
ssh-keygen																								#在本地生成密钥对
cat .ssh/id_rsa.pub 																			#将公钥复制进github-setting-sshandGPG keys
git push -u origin master																	#把本地master代码上传到github的origin库
git clone git@github.com:zlwm1152/linux-site.git					#把远程仓库的代码克隆到本地当前目录
```

## 五、Gitlab备份