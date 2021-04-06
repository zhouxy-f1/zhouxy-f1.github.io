##### Vim规律

```shell
#定位相关
G		代表尽头
$		代表行尾
^		代表行首
w		代表下一个单词
b		上一个单词
e		当前单词末尾

#操作
y		代表复制
d		代表删除
p		代表粘贴
.		代表重复命令
h、l、j、k			左右下上
w、b					 移动到下|上一个单词
```

## 普通模式

```shell
#定位
gg				跳到首行首字母
G					跳到末行首字母
nG|ngg		跳转到第n行

f+字母		向后搜索'字母' 并跳转到第一个匹配的位置
F+字母		向前搜索'字母' 并跳转到第一个匹配的位置

#翻页
ctrl+f		向下翻页
ctrl+b		向上翻页

#删除|剪切
x			反退格键（delete键） 
X			退格键
dw		删除一个单词（不适用中文）类似于上一节说的cw，只是删除后还在普通模式
d$		删除至行尾
d^		删除至行首
D			删除至行尾
C			删除至行尾并进入编辑模式
dG		删除到文档结尾处 
d1G		删至文档首部
ddp		交换上下行

#复制
yy		复制游标所在的整行（3yy表示复制3行）
y|y0	复制至行首，不含光标所在处字符。
y$		复制至行尾。含光标所在处字符。
yw		复制一个单词。
y2w		复制两个单词。
yG		复制至文本末。
y1G		复制至文本开头

#粘贴
p(小写)代表粘贴至光标后（下）
P(大写)代表粘贴至光标前（上）

#撤销

u				撤销上次操作
ctrl+r	回滚，取消上次的撤销
r				替换当前光标标记的单个字符
R				进入REPLACE模式, 连续替换，ESC结束

#快速退出
Shift+zz	即可保存退出vim


#单词补全
字母+crtl+p

#数字增减
+1  ctrl+a
+n	n+ctrl+a
-1  ctrl+x
-n	n+ctrl+x

#不退出vim执行shell命令
:! shell命令

#快速复制粘贴指定行到目标行
:起始行,终止行 t 目标行
:起始行,终止行 m 目标行
```

## 进入|退出编辑模式

```shell
i			在当前光标处进行编辑
I|A		在行首/末插入
a			在光标后插入编辑
o|O		在当前行后/前插入一个新行
cw		删除一个单词，同时进入插入模式
ESC		退出编辑模式
```

## 末行模式

```shell
#搜索
/string		搜索string ，n 向下依次搜索，N 向上依次搜索 
:number		跳到第n行
/noh			取消搜索高亮

#替换
:1,5s#sbin#test#g          替换1-5行中包含sbin的内容为test
:%s#sbin#test#g            替换整个文本文件中包含sbin的替换为test
:%s#sbin#test#gc           替换内容时时提示是否需要替换
%表示所有行   s表示替换   g表示所有匹配到的内容     c表示提示

#替换为 w (y/n/a/q/l/^E/^Y)？
y:替换一次
n:选中下一个
a:全部替换
q:退出
l:替换一次并退出询问

#读入
:r  /etc/hosts 		读入/etc/hosts文件至当前光标下面
:5r /etc/hosts 		指定/etc/hosts文件当前文件的哪行下面另存

#保存|另存
:x	 先保存再退出
:w	 保存不退出
:w /root/test			将文件所有内容另存为/root/test
```

## 可视模式

```shell
#可视块模式
ctrl+v 	进入可视块模式
1.插入:按shift+i进入编辑模式,输入#,结束按ESC键
2.删除:选中内容后，按x或者d键删除
3.替换:选中需要替换的内容, 按下r键,然后输入替换后的内容

#可视行模式
shift+v  进入可视行模式
1.选择：方向上下方向键选择行
2.复制：选中行内容后按y键及可复制。
3.删除：选中行内容后按d键删除。
4.调整为bash排版：选中所有行，再按=
```

## 环境变量

```shell
#临时环境变量
:set nu		临时显示行号
:set ic		临时忽略大小写, 在搜索的时候有用
:set ai		临时自动缩进
:set list	临时显示制表符(空行、tab键)
:set no[nu|ic|ai…]		临时取消临时设定的变量
:noh								临时取消搜索高亮
#永久环境变量
[root@m01 ~]# vim /etc/vimrc	（全局）		vim ~/.vimrc(个人)
syntax on						语法检查及高亮
set fenc=utf-8			设定默认解码
set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936
set number					显示行号
set autoindent			vim使用自动对齐，也就是把当前行的对齐格式应用到下一行
set smartindent			依据上面的对齐格式，智能的选择对齐方式
set tabstop=4				设置tab键为4个空格
set shiftwidth=4		设置当行之间交错时使用4个空格
set ruler						设置在编辑过程中,于右下角显示光标位置的状态行
set incsearch				设置增量搜索,这样的查询比较smart
set showmatch				高亮显示匹配的括号
set matchtime=10		匹配括号高亮时间(单位为?1/10?s)
set ignorecase			在搜索的时候忽略大小写
set nobackup				禁止生成临时文件

set cursorline			当前行高亮
set t_Co=256				支持256色
colorscheme molokai	使用molokai颜色模板
```

## vim拓展

```shell
#如何同时编辑多个文件
vim -o file1 file2	水平分割
vim -O file1 file2	垂直分割
ctrl+ww							文件间切换

#相同文件之间差异对比
diff  文件对比(用的不多)    
vimdiff  以vim方式打开两个文件对比,标记不同的内容

#非正常退出报错
如果VIM非正常退出 （ctrl+z）挂起或强制退出终端没关闭VIM后，只需要删除同文件名的.swp文件即可解决
.filename.swp   rm -f .filename.swp
```

##### vim配置文件

```bash
vim ~/.vimrc

set history=1000
set autoread
set bg=light

set tabstop=4   4个空格为一个tab

set foldenable
set foldmethod=manual

set fenc=utf-8
set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936
set showmatch
set matchtime=10
set ignorecase
set smartcase
set incsearch
```





























































