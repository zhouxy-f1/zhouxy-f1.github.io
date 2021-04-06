

```bash
yml是一种标记语言，强调以数据为中心

#基本语法
1.缩进时不允许用tab键，只允许用空格
2.缩进的空格数目不重要，但要求相同层级的元素左侧要对齐
3.用井号做注释，从井号开始到行尾都会被忽略


#yml支持的数据结构
1.对象：键值对的集合，又叫映射（mapping）、哈希（hashes）、字典(dictionary)
name: zhouxy
age: 18
hash: {name: zhouxy,age: 18}

2.数组：一组按次序排列的值，又称序号（sequence）、列表（list），用连词线开头的行表示
animal:
- Cat
- Dog
animal: [Cat,Dog] 

3.纯量：单个的不可分割的值，即（scalars），比如字符串、布尔值、整数、浮点数、Null（用~表示）、时间、日期等
number: 5.22
isSet: true
child: ~
iso8601: 2020-12-25t21:59:59.10-5:00
date: 1988-05-22												#日期可采用复合iso8601格式 
e: !!str 123														#用!!强制转换数据类型
f: !!str true

#字符串
str: sdfjsodgfdsg						#在yaml语法中字符串不需要用引号
str: 'you are so newb'			#但是当字符串中有空格的，需要用单引号括起来
str1: 'like\nyou'						#yaml语法中双引号是强引用，单引号是弱引用。单引号会把\n转义为换行符
str2: "like\nyou"						#双引号不会对特殊字符转义
str: 'father''s day'				#'s需要在前面加个'来转义，结果是father's day
str: |-											#+表示保留行尾的换行符，-表示删除行尾的换行符；|也可用>代替
  这串字符太长了 
  需要写成两行


1.字符串太长的话可写成多行，从第二行开始，必须有一个单空格缩进，换行符会转为空格
示例：
str: 好长的字符串啊，写
 不下了，换个行
 吧
2.多行字符串可以使用|保留换行符，也可以使用>折叠换行
示例
this: |
Foo
Bar
that: >
Foo
Bar
3.+表示保留文字块末尾的换行，-表示删除字符串末尾的换行
示例
s1: |
  Foo
s2: |+
  Foo
s3: |-
  Foo
```

