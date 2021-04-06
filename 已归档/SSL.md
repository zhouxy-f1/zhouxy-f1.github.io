## 加密算法

##### 基本概念

```bash
#对称加密
加密和解密都用同一个密钥，其加解密过程是：
明文+加密算法+私钥=>密文
密文+加密算法+私钥=>明文
常见加密算法：DES、3DES、TDEA、Blowfish、RC5、IDEA、AES-GCM、ChaCha2--Poly1305等
特点：算法公开，加密解密速度快，适用于对大量数据加密，但安全性不高，一旦密钥泄露，通讯系统全盘凉凉。

#非对称加密
使用公钥加密，私钥解密，其公钥和算法是公开的但私钥是保密的，需要用时把公钥发给别人来加密数据，自己用私钥对密文解密。
其加解密过程是：
明文+加密算法+公钥=>密文
密文+解密算法+私钥=>明文
常见算法：RSA、DSA、ECDSA、DH、ECDHE等
特征：这种算法性能较低但安全性超强

#哈希算法
将任意长度的信息转换为较短的固定长度的值，通常其长度要比信息小得多，且算法不可逆。
例如：MD5、SHA-1、SHA-2、SHA-256 等

echo -n '123'|md5sum									#对字符串加密，得到md5值：202cb962ac59075b964b07152d234b70  -
md5sum 123.txt >/opt/123.md5					#对文件加密，123.md5里保存的是md5值
md5sum -c /opt/123.md5								#对md5值校验，返回 123.txt OK；如果对123.txt有修改则提示FAILED

#数字签名
签名就是在信息的后面再加上一段内容（信息经过hash后的值），可以证明信息没有被修改过。hash值一般都会加密后（也就是签名）再和信息一起发送，以保证这个hash值不被修改。
```

## https

##### 相关协议

```bash
HTTP协议（HyperText Transfer Protocol，超文本传输协议）：是客户端浏览器或其他程序与Web服务器之间的应用层通信协议
HTTPS协议（ xxx over Secure socket layer）：HTTP+SSL/TLS，即HTTP下一层加入SSL层，HTTPS的安全基础是SSL。
SSL（Secure Socket Layer,安全套接字层）：位于TCP/IP层与应用层之间，为数据通讯提供安全支持
TLS（Transport layer security，传输层安全）：前身是SSL，目前广泛使用TLS1.1和TLS1.2
CSR文件是您的公钥证书原始文件，包含了您的服务器信息和您的单位信息，需要提交给CA认证中心审核
```

##### https原理

```bash
#证书验证阶段（非对称加密）
1.当用户通过浏览器对客户端80端口发起连接请求，此请求携带了浏览器支持的加密算法和哈希算法
2.服务器跳转到443端口，把证书发给浏览器
3.浏览器判断证书是否合法，如果不合法则告警提示，但用户点击访问这个不安全的网站依然可以访问

#数据传输阶段（对称加密）
3.浏览器生成随机数R，用公钥加密R发给服务端，服务端用私钥解密得到明文的R
4.服务端用随机数R构造对称加密算法，通过此算法与浏览器进行密文通讯

#浏览器如何检验证书合法性（通过内置的TLS来完成）
1.浏览器收到客户端发来的证书后，先在内置的证书列表中找到对应的发证机构，如果没有则提示用户：“不是权威机构颁发”
2.如果对到了对应机构，则提取该机构的公钥来解密证书，得到证书签名和证书内容（网址、有效期等）
3.浏览器依次验证证书数字签名的合法性 =>证书记录的域名是否匹配当前域名=>证书有效期
即便黑客使用CA认证了的证书，通过证书的验证，也无法用私钥解密浏览器发过来的随机数，因为私钥跟证书是配对产生的
```

##### 申请CA证书

```bash
由于https仅解决了通讯安全的问题，黑客可通过命令：openssl genrsa -idea -out server.key 2048来自己生成证书和密钥，劫持用户请求，把自己的证书发给浏览器，从而跟客户端建立通讯，获取客户端发送过来的数据如账户、密码、银行卡号等敏感信息，所以需要权威机构CA中心颁发证书

#如何申请证书
先去机构登记信息，登记机构把信息通过CSR发给CA，CA中心审核后，生成私钥和证书，公钥保存在CA证书链中。

以阿里为例，步骤如下：
1.购买证书
进入阿里云SSL证书模块-购买证书-选择云盾证书服务（单域名，DV SSL 免费版）-立即购买
2.提交证书申请资料
填写绑定的域名、添加一条阿里提供的解析记录到DNS，选择由系统自动生成csr文件
3.阿里云可以直接把ssl证书绑定到SLB，也可以手动添加到ECS服务器，需要下载证书（*.key和*.pem两个文件）

#多域名版ssl证书分3种验证等级
1.域名型DV
权验证域名所有权，适用于个人站点，申请需10分钟-24小时，有效期1年，免费保护3个域名，超过部分每个280元/年
2.企业型OV
增加有全面的企业身份验证适用于中小企业及电子商务平台；审核周期1-3天，有效期1-2年，免费保护4个域名，超过部分1800元/年
3.增强型EV
增加有最高等级的企业身份验证，适用于金融机构、政府等，在绿色地址栏显示企业名称，审核3-5天有效1-2年，保护3个域名

#注意事项
https不支持续费，到期需要重新申请并替换原证书
https不支持三级域名，如source.m.longines.com
https显示黄色代表网站代码中包含http的不安全链接
https显示红色代表证书验证失败，要么假证书要么证书过期
```

##### 证书相关操作

```bash
#证书相关文件
.key文件: 是证书私钥文件，如果申请证书时没有选择系统创建CSR，则没有该文件。请您保存好该私钥文件。
.pem文件: 是证书文件，pem文件的扩展名为crt,Nginx证书会使用扩展名文件，在阿里云SSL证书中与.crt文件一样。
.pfx文件: 一般适合Tomcat/IIS服务器。每次下载都会产生新密码，该密码仅匹配本次下载的证书。如需更新证书，也要更新密码
.der文件: 一般出现在java平台
.crt文件: 是证书文件，一般包含两段内容，如
-----BEGIN CERTIFICATE-----
***服务器证书
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
***CA中间证书
-----END CERTIFICATE-----
如果是Apache服务器，会将证书文件拆分成 _public.crt（证书）文件和_chain.crt（证书链或中间证书）文件。

#查看证书信息
openssl x509 -in xxx.crt -noout -dates					#x509是给CA颁发的证书专用 -in指定证书 -noout不输出
-dates				查看证书过期时间
-text					查看证书内容
-serial				查看证书序列号
-subject			查看拥有者名字
-fingerprint	查看md5指纹

#证书格式转换
mv zlwm115.com.crt zlwm115.com.pem																	#cer和crt格式直接改名即可
openssl pkcs12 -in certname.pfx -nocerts -out key.pem -nodes				#pfx格式转pem,提取私钥
openssl pkcs12 -in certname.pfx -nokeys -out cert.pem								#pfx格式转pem,提取证书
openssl x509 -inform der -in certificate.cer -out certificate.pem		#der转pem,提取证书
openssl rsa -inform DER -outform PEM -in privatekey.der -out privatekey.pem	#der转pem，提取私钥


#判断证书与私钥是否匹配
openssl x509 -pubkey -noout -in www.zlwm115.com.crt
openssl rsa -pubout -in www.zlwm115.com.key
```

##### 配置ssl

```bash
见md文件：架构服务_LNMP
```







![5de87413bcb01](https://tva1.sinaimg.cn/large/007S8ZIlgy1gficud9ehcj30go0fc3z3.jpg)













