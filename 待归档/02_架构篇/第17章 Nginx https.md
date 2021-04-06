## 为什么要用https

##### 原因

数据通过明文传输，不安全，易被劫持篡改

##### DNS劫持，篡改演示





## 证书申请流程

```shell
我们首先需要申请证书，需要进行登记（我是谁，什么组织，我想做什么等）到了登记机构在通过CSR发给CA，CA中心通过后，CA中心会生成一对公钥和私钥，那么公钥会在CA证书链中保存，公钥和私钥证书订阅人拿到后，会将其部署在WEB服务器上
```

![WX20191021-121058](https://tva1.sinaimg.cn/large/006y8mN6gy1g85p48cputj30so0hgq8f.jpg)

## ssl证书类型

![image-20191021121427693](https://tva1.sinaimg.cn/large/006y8mN6gy1g85p6o681zj30sw0dwjvz.jpg)

##### HTTPS证书购买选择

保护1个域名 www
保护5个域名 www images cdn test m
通配符域名 *.oldboy.com

##### HTTPS注意事项

Https不支持续费,证书到期需重新申请新并进行替换.
Https不支持三级域名解析, 如test.m.oldboy.com

##### 颜色指示

Https显示绿色, 说明整个网站的url都是https的。
Https显示黄色, 因为网站代码中包含http的不安全连接。
Https显示红色, 要么证书是假的，要么证书过期。

