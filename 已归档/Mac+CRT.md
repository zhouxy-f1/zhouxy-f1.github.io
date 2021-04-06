## MAC

##### 系统优化

```bash
#修改主机名
hostname													#查看主机名
hostname Mac											#临时修改主机名
scutil --set HostName MacBookAir	#永久修改主机名

#修改命令提示符
mac的root用户和普通用户的环境变量配置在不同文件里，
1.改root命令提示符
vim /etc/bashrc
PS1='[\u@\h \W]\$ '
source  /etc/bashrc

2.改普通用户的PS1，用普通用户登陆
sudo vim /etc/zshrc
PS1="[%n@%m %1~]%$ "							#n是用户名 m是主机名 ~是相对路径 d是绝对路径 D是系统日期 T是系统时间
 source /private/etc/zshrc


#改环境变量，普通用户也一样改
cat>.zshrc<<EOF
alias ll='ls -lhB'
EOF

#mac连vpn
设置-网络-点+号（接口：vpn ，vpn类型：Cisco IPSec）-（服务器地址：180.167.200.178 账户名 jonny.zhou 密码xiaoyong）-认证设置（共享的密钥：1sBAfuGBRu3Z  群组名称：ChinaNetCloud）
#iPhone连vpn
设置-通用-VPN-添加vpn配置-（类型：IPsec 服务器：180.167.200.178 账户：jonny.zhou 密码：xiaoyong 群组名称：ChinaNetCloud 密钥：1sBAfuGBRu3Z）-完成-连接即可

#usb断电
sudo killall -STOP -c usbd
```

##### 软件安装

```bash
在mac上的软件管理工具是homebrew，参考官网：www.brew.sh

#下载brewhome
1.下载安装homebrew的脚本，可通过在线安装，如果网络不好可从个人百度网盘下载，再本地安装
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"

链接: https://pan.baidu.com/s/1PnK0O9nHZyg0e6gYUi-u3Q 提取: 4k2x		#下载brew_install.rb
原：BREW_REPO = "https://github.com/Homebrew/brew".freeze					 #需要换下载brewhome的源
换：BREW_REPO = "https://mirrors.ustc.edu.cn/brew.git".freeze

2.通过下载下来的脚本来安装brewhome
ruby /Users/zhouxy/Downloads/brew_install.rb											 #本地执行脚本来安装brewhome

3.上面的脚本安装好brewhome后会下载homebrew-core,但core的源依然是国外的，可用命令换成中科院的源来下载
git clone git://mirrors.ustc.edu.cn/homebrew-core.git/ /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core --depth=1

4.更换brewhome-core和cask的源
cd "$(brew --repo)"
git remote set-url origin https://mirrors.ustc.edu.cn/brew.git
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core" 
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
cd "$(brew --repo)/Library/Taps/caskroom/homebrew-cask"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile

5.安装完成之后检查brewhome状况
brew -v
brew update
brew doctor

6.卸载brewhome
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"

#homebrew的基本用法
brew update					#更新Homebrew
brew list						#列出已安装的软件包
brew upgrade				#更新所有安装过的软件包
brew upgrade wget		#更新指定的软件包
brew search wget		#查找软件包
brew install wget		#安装软件包
brew remove wget		#卸载软件包

brew info wget			#查看软件包信息
brew deps wget			#列出软件包的依赖关系
brew outdated				#列出可以更新的软件包	
brew doctor					#检查brew是否有问题
brew inistall nginx								#安装nginx
brew install elasticsearch@5.6		#如果search结果多个版本，可用@指定版本
brew services start nginx					#启动nginx
brew services stop nginx					#停止nginx
brew services restart nginx				#重启nginx
brew services list								#列出brew当前的状态
```

## CRT8.3

```bash
#记录日志
Preferences-General-Default-Edit Default Setting-Log file
Log file name :/Users/Shared/学习/CRT_logs/%Y-%M-%D_%S.log
Options:Start log upon connect , Append to file,Start new ...
upon connect:[%Y-%M-%D %h:%m]Login
upon disconnect:[%Y-%M-%D %h:%m]Logout

#显示字体
Termianl-Appearance
Current color scheme:Monochrome
Fonts:Courier New 15p

#快捷键输入字符
Edit Default Option-Mapped Keys-Map a key - 输入快捷键 - 输入string\r-确定

#按单词移动光标
点Edit Default Option - 点Mapped Keys - 按command + 右方向键 - 输入 \033f 
点Edit Default Option - 点Mapped Keys - 按command + 左方向键 - 输入 \033b

#屏显缓存
在Session Option或Edit Default Option里：Terminal-Emulation-Scrollback buffer :50000（最大125000）

#双击标签页克隆
在Global Option里:Terminal-Tabs/Tiling-Options-Double click action (Clone Tab)

#底部按钮
先显示 view-Buttion Bar 然后配置 在底部右键-New Butter


#彻底清除crt配置
rm -rf  ~/Library/Application/Support/VanDyke/
rm -fr /Users/zhouxy/Library/Application Support/
#迁移crt配置文件
保存整个config目录，替换到新crt上对应的目录即可，该目录位于
/Users/zhouxy/Library/Application Support/VanDyke/SecureCRT/Config
```

