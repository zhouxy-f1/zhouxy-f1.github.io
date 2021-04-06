## Shell常识

```shell
#shell用途
01）基础配置：系统初始化操作、系统更新、内核调整、网络、时区、SSH优化等。
02）安装程序：LNMP、LAMP、MySQL、Nginx、Redis等。
03）配置变更：Nginx Conf、PHP Conf、MySQL Conf、Redis Conf等。
04）业务部署：Shell配合Git、Jenkins实现自动化部署PHP、Java代码，以及代码回滚。
05）日常备份：MySQL全备 + 增量 + binlog + crond + Shell脚本备份等。
06）信息采集：Zabbix + Shell: 对硬件、系统、服务、网络的监控等。
07）日志分析：ELK：取值->排序->去重->统计->分析等。
08）服务扩容/缩容：Zabbix + Shell

扩容: 监控服务器cpu, 如cpu负载持续80% + 触发动作(脚本)
脚本: 调用api开通云主机->初始化环境->加入集群->对外提供访问    
缩容: 监控服务器cpu使用率20%->判断有多少web节点->判断是否超过预设->缩减到对应的预设状态->变更负载的配置

#书写规范
1.统一存放目录 			 例如：/scripts/
2.使用幻数指定解释器，例如：#!/bin/bash 或 #!/usr/bin/python 或 #!/usr/bin/env python
3.附带作者及版权信息，例如：#此脚本作用 By:zhouxy  At:2020-3-31
4.尽量写绝对路径

#判断是否交互式shell
用命令echo $-，结果有：himBH 或 hB，有i的是交互式shell
解释如下
h：hashall，打开这个选项后，Shell会将命令所在的路径记录下来，避免每次都要查询
i：interactive，包含这个选项说明当前的Shell是一个交互式的Shell
m：monitor，打开监控模式，就可以通过Job control来控制进程的停止、继续，后台或者前台执行等
B：braceexpand，大括号扩展
H：history，Shell会把我们执行的命令记录下来，可以通过history命令查看

#脚本执行顺序
登陆式shell每个调用的脚本会依次撤销前一个调用脚本中的改变，在退出登录Shell时，我们还可以执行某些任务，如创建自动备份、清除临时文件。把这些任务放在.bash_logout文件中。
/etc/profile
/etc/profile.d/*.sh
./bash_profile
./bashrc
/etc/bashrc

非登陆式shell
~/.bashrc
/etc/bashrc
/etc/profile.d/*.sh

#脚本执行方式可用以下方式，如需直接用路径来执行，需要先加执行权限
sh|source test1.sh
cat test1.sh|bash
bash<test1.sh
chmod +x test1.sh    =>  /script/test1.sh
```

## 变量

```bash
#定义变量
命名：由字母、数字、下划线几个组成，不可以数字开头，变量名最好具备一定的含义
1.不能出现"-横岗"命令
2.等号两边不能有空格，需要有空格时，必须使用引号
3.定义的变量不要与系统命令出现冲突

#位置变量
$0 这个程式的执行名字
$n 这个程式的第n个参数值，n=1..9
$* 这个程式的所有参数,此选项参数可超过9个。
$# 这个程式的参数个数
$$ 这个程式的PID(脚本运行的当前进程ID号)
$! 执行上一个背景指令的PID(后台运行的最后一个进程的进程ID号)
$? 执行上一个指令的返回值 (显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误)
$- 显示shell使用的当前选项，与set命令功能相同
$@ 跟$*类似，但是可以当作数组用

#变量匹配
${#FILE_NAME}	获取变量值的长度
${变量#匹配规则}	从头开始匹配，最短删除
${变量##匹配规则}	从头开始匹配，最长删除
${变量%匹配规则}	从尾开始匹配，最短删除
${变量%%匹配规则}	从尾开始匹配，最长删除
${变量/旧字符串/新字符串}	替换变量内的旧字符串为新字符串，只替换第一个
${变量//旧字符串/新字符串}	替换变量内的旧字符串为新字符串，全部替换
${变量:匹配规则:匹配规则}	索引及切片

#变量切割
例：url=www.sina.com.cn
${url:0:5}  //从0列开始，切出5列，第0-4列，结果为：www.s
${url:5:5}	//从5列开始，切出5列，第5-9列，结果为：ina.c
${url:5} 		//去除前5列，结果为：ina.com.cn

#变量运算
expr $a + $b		//加号两边要有空格，否则输出7+8
echo $[$a+$b]		//不支持小数运算
echo $(($a+$b))	//不支持小数
echo $a+$b|bc		//支持小数运算

#变量自增减
a=1;let a++;echo $a 

#示例
file=/dir1/dir2/dir3/my.file.txt

${file#*/}		删掉第一个 / 及其左边的字符串：dir1/dir2/dir3/my.file.txt
${file##*/}		删掉最后一个 /  及其左边的字符串：my.file.txt
${file#*.}		删掉第一个 .  及其左边的字符串：file.txt
${file##*.}		删掉最后一个 .  及其左边的字符串：txt
${file%/*}		删掉最后一个  /  及其右边的字符串：/dir1/dir2/dir3
${file%%/*}		删掉第一个 /  及其右边的字符串：(空值)
${file%.*}		删掉最后一个  .  及其右边的字符串：/dir1/dir2/dir3/my.file
${file%%.*}		删掉第一个  .   及其右边的字符串：/dir1/dir2/dir3/my

记忆的方法为：
# 是 去掉左边（键盘上#在 $ 的左边）
%是去掉右边（键盘上% 在$ 的右边）
单一符号是最小匹配；两个符号是最大匹配
${file:0:5}：提取最左边的 5 个字节：/dir1
${file:5:5}：提取第 5 个字节右边的连续5个字节：/dir2
```

## 条件语句

```bash
#if语句格式
if [ 条件1 ];then
	语句1
elif [ 条件2 ];then
	语句2
elif [ 条件3 ]
	语句3
else
	语句4
fi

#case条件格式
case $KEY_STATUS in
	stat)	echo "$STATUS"|grep -A1 "$NODE_NAME"|tail -1 ;;
	rise)	echo "$STATUS"|grep -A2 "$NODE_NAME"|tail -1 ;;
	fall)	echo "$STATUS"|grep -A3 "$NODE_NAME"|tail -1 ;;
	type)	echo "$STATUS"|grep -A4 "$NODE_NAME"|tail -1 ;;
	*)		echo "";;
esac

#if条件之文件比较
例：[ -d|e|f| file ]，可通过判断pid文件是否存在来看服务是否在运行
-e	如果文件或目录存在则为真
-d	如果目录存在则为真
-f	如果普通文存在则为真
-s	如果文件存在且至少有一个字符则为真
-r	如果文件或目录存在且可读则为真
-w	如果文件或目录存在且可写则为真
-x	如果文件存在且可执行则为真
实例：[ -d /etc/ ] && echo "目录存在" || echo "目录不存在"

#if条件之整数比较
-eq	等于则条件为真			  --equal
-ne	不等于则条件为真		 --notequal
-gt	大于则条件为真			  --greatthan
-lt	小于则条件为真			  --lessthan
-ge	大于等于则条件为真		 --great&equal
-le	小于等于则条件为真		 --less&equal

实例：根据学生录入的成绩判断等级
#!/bin/bash
[root@qiudao ~/shell]# cat if-7.sh
read -p "请输入你的分数: " fs
if [ $fs -gt 0 -a $fs -lt 60 ];then
    echo "补考"
elif [ $fs -ge 60 -a $fs -lt 80 ];then
    echo "合格"
elif [ $fs -ge 80 -a $fs -le 100 ];then
    echo "优秀"
else
    echo "顽皮"
fi

#if条件之字符比较
字符需要用单引号或双引号引起来，且运算符左右需要空格；变量不需要引号，且运算符左右不需要空格
==	等于则条件为真 	["a" == "b"]
!=	不相等则条件为真	["a"!="b"]
-z	字符串的长度为零则为真[ -z $a ]

a=123;[ -z $a ]&& echo "空" ||echo "非空"			>>非空
a="";[ -z "$a" ]&& echo "空" ||echo "非空"			>>空
a=台湾;b=中国;[ $a==$b ]&& echo "$a是$b的"||echo "$a不是$b的"

#if条件之正则比较
正则比较时需要用[[ ]]，多条件时用&&（逻辑且）,||(逻辑或)
例：
[[ $USER =~ ^r ]]&& echo "正则"
[[ $n =~ ^[0-9]+$ ]]														//判断是否正整数
[[ $name =~ ^[a-Z]+ && $name =~ [0-9]+$  ]]			//判断名字以字母开头且数字结尾
[[ $1 =~ ^[^0-9]+$ || $3 =~ ^[^0-9]+$ ]]				//正则里^号取反
! [[ $2 =~ [*/%+-] ]];then											//括号外整体取反 
```

## 循环语句

```bash
#for循环格式
for i in $(seq 3)
do
		echo $i
done
-------------------------
for ((i=0;i<10;i++))
do
	commands
done
-------------------------
while [条件] 或 true 或 read line
do
		循环体
done<username.txt
------------------------

取值方法
for var in file1 file2 file3										 	#直接取in后面的值，默认以空格做分隔符
list="file1 file2 file3"；for var in $list			   #从变量取值，存入变量
for var in `cat /etc/hosts`;do echo $var;done  	  #从文件中取值，默认以空格、换行、tab为分隔

IFS=:							#以冒号为分隔符，从文件中取值时修改分隔符,for循环之前定义一个IFS变量即可
IFS=:;"						#以冒号、分号、双引号为分隔符
IFS=$'\n'					#以换行符为分隔符

处理特殊符号
for var in file1 "fi le2" 'fi le3' file\'4				#有单引号时，必须用\转义；有空格要的引号，否则语法错误

变量自增
i=4;a=i++					#此时，先做a=i=4，再做i++，故a=4，i=5		
i=4;a=++i 				#此时，先做++i=5，再做a=i，故a=5, i=5

for ((i=19;i<24;i++))
do
        date -s "2019-10-$i" &>/dev/null
        touch /opt/`date +%F` &>/dev/null
done

#while循环格式
while [条件] 或 true 或 read line
do
		循环体
done<username.txt

read line表示一行一行地读取done后面username.txt的内容，每一行内容用$line来调用


#跳出循环
exit			//退出整个程序
break			//跳出当前循环
continue 	//忽略本次循环剩余代码，进入下次循环

PWD=$(echo $RANDOM|md5sum|cut -c 5-15)
echo $PWD |passwd --stdin $line &>/dev/null

```

## 函数

```bash
#简单示例
check(){																#定义函数，或用function check(){cmds}
        echo "check infomation"
        echo "the second cmd"						#命令后不用符号
}
check																		#调用函数，直接写函数名即可
#传参
check(){
        echo "$1"
}
check first	second						#调用函数时后面加参数，$1表示第1个参数，$*表示所有参数,参数间空格隔开
#返回值
count(){
        echo 200
        return 25							#return后接0-255间的整数，表示返回状态，用$?调用。不指定则$?是0表执行成功
}
result=`count`
echo "result of return is $?"						#return返回状态后直接退出，后面命令不再执行
echo "result of echo is $result"
```

## 数组

```bash
数组也是变量，1个数组可存多个值；普通数组只能用0-n的整数为索引，关联数据可用字符串为索引，用declare -A来声明

普通数组
fruit=([0]=lemon [1]=apple [2]=banana [3]=orange)			#无须定义直接赋值，这样一次性赋值会覆盖原有的所有值
echo ${fruit[3]}								#调用数组
echo ${fruit[*]}								#查看所有索引的值。[*]表示所有索引
echo ${fruit[*]:0}							#查看第0号索引及后面所有索引的值
echo ${fruit[*]:1:2}						#查看第1号索引及后面2个索引的值
set |grep fruit									#查看所有索引及对应的值
unset fruit											#取消定义的char数组
echo ${!fruit[*]}								#查看所有索引,返回0123
echo ${#fruit[*]}								#统计索引数量,返回4

关联数组
declare -A PE															#一个关联数组。必须先声明，否则会被当成普通数组
PE=([name]=zhouxy [gender]=male [age]=18)	#一次性赋值会覆盖原有的值
echo ${PE[name]}													#调用数组，方法与普通数组一样
unset PE																	#取消定义

遍历普通数组
while read line
do
	let i++
	pass[i]=$line
	echo "索引${i}的值是：${pass[i]}"
done</etc/passwd

遍历关联数组
declare -A passwd
while read line
do
	shell=$(echo $line|awk -F: '{print $NF}')
	let passwd[$shell]++
done</etc/passwd

for i in ${!passwd[*]}
do
	echo $i ${passwd[$i]}
done
```

