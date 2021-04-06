```bash
#Robots简介
Robots Exclusion Protocol，有时候有些页面访问消耗性能比较高不想让搜索引擎抓取，可以在根目录下放robots.txt文件屏蔽搜索引擎或者设置搜索引擎可以抓取文件范围以及规则。

文件写法：
User-agent: 这里的代表的所有的搜索引擎种类，*是一个通配符:
Disallow: /admin/ 这里定义是禁止爬寻admin目录下面的目录:
Disallow: /require/ 这里定义是禁止爬寻require目录下面的目录:
Disallow: /ABC/ 这里定义是禁止爬寻ABC目录下面的目录:
Disallow: /cgi-bin/*.htm 禁止访问/cgi-bin/目录下的所有以".htm"为后缀的URL(包含子目录):
Disallow: /? 禁止访问网站中所有包含问号 (?) 的网址:
Disallow: /.jpg$ 禁止抓取网页所有的.jpg格式的图片:
Disallow:/ab/adc.html 禁止爬取ab文件夹下面的adc.html文件:
Allow: /cgi-bin/　这里定义是允许爬寻cgi-bin目录下面的目录:
Allow: /tmp 这里定义是允许爬寻tmp的整个目录:
Allow: .htm$ 仅允许访问以".htm"为后缀的URL:
Allow: .gif$ 允许抓取网页和gif格式图片:
Sitemap: 网站地图 告诉爬虫这个页面是网站地图:

例1. 禁止所有搜索引擎访问网站的任何部分:
User-agent: *
Disallow: /
例2. 允许所有的robot访问 (或者也可以建一个空文件 “/robots.txt” file)：
User-agent: *
Allow:　/
例3. 禁止某个搜索引擎的访问:
User-agent: BadBot
Disallow: /
例4. 允许某个搜索引擎的访问:
User-agent: Baiduspider
allow:/

#判断一个ip是否爬虫
例：151.221.196.151这个ip非爬虫；120.66.125.123是百度爬虫
[root@MacBook-Air ~]# host 139.196.221.151
Host 151.221.196.139.in-addr.arpa. not found: 3(NXDOMAIN)
[root@MacBook-Air ~]# host 123.125.66.120
120.66.125.123.in-addr.arpa domain name pointer baiduspider-123-125-66-120.crawl.baidu.com.



https://www.longines.cn/robots.txt
User-agent: *
Disallow: /skins/
Disallow: /js/
Disallow: /upload/
Disallow: /media/
Disallow: /static/
Disallow: /catalog/product_compare/
Disallow: /ajaxcomparator/index/toggle/
Disallow: /catalogsearch/ajax/suggest/
Disallow: /catalogsearch/
Disallow: /customer/section/load/
Disallow: /media/catalog/
Disallow: /watches/selector
Sitemap: https://www.longines.cn/longines_sitemap.txt

geo $limited {
default 0;
#Customer IP
116.196.103.0/24 1;
180.173.17.210 1;
#Baidu
123.125.67.0/24 1;
#msn
40.77.167.0/24 1;
#Sogou
#other
116.6.40.0/24 1;
# 871423 customer asked to limit
123.206.225.251/32 1;
123.206.191.189/32 1;
```

