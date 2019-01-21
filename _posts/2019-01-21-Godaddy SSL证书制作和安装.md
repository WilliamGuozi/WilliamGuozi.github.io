# SSL：GoDaddy SSL证书制作和安装

## 简介  
> SSL证书是数字证书的一种，类似于驾驶证、护照和营业执照的电子副本。因为配置在服务器上，也称为SSL服务器证书。SSL 证书就是遵守SSL协议，由受信任的数字证书颁发机构CA，在验证服务器身份后颁发，具有服务器身份验证和数据传输加密功能。
 
> 阿里云SSL证书的配置非常简单，申请完成，打包下载放到你的web服务器上即可；与阿里云上的SSL证书不同，GoDaddy作为一个专门的域名和SSL证书服务的提供商，其为保证SSL证书的安全性，整个证书的操作过程都需要自己来做，本篇文章就主要介绍在GoDaddy上SSL证书的制作和安装，同时介绍一下SSL的工作原理，希望对需要的小伙伴有所帮助。

## SSL证书的工作原理

> 证书主要作用是在SSL握手中，我们来看一下SSL的握手过程

1. 客户端提交https请求
2. 服务器响应客户，并把证书公钥发给客户端
3. 客户端验证证书公钥的有效性
4. 有效后，会生成一个会话密钥
5. 用证书公钥加密这个会话密钥后，发送给服务器
6. 服务器收到公钥加密的会话密钥后，用私钥解密，回去会话密钥
7. 客户端与服务器双方利用这个会话密钥加密要传输的数据进行通信
![img-w500](/images/201901211728.png) 

## GoDaddy 证书制作过程  

### 制作CSR证书签署请求文件
> 生成证书签署请求CSR(Certificate Signing Request)文件，本文以glinux.top域名为例，需要填写的信息中，请注意Common Name，应为泛域名地址，如: *.glinux.top  

```openssl req -new -newkey rsa:2048 -nodes -keyout glinux.top.key -out glinux.top.csr```

```
Generating a 2048 bit RSA private key
.......................+++
...................+++
writing new private key to 'glinux.top.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:ZhejiangSheng
Locality Name (eg, city) [Default City]:Hangzhou
Organization Name (eg, company) [Default Company Ltd]:glinux
Organizational Unit Name (eg, section) []:DevOps
Common Name (eg, your name or your server's hostname) []:*.glinux.top
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
> 生成两个文件，glinux.top.csr为证书签署请求，glinux.top.key为解密使用的私钥  

![img-w500](/images/201901211759.png) 

> 将证书签署请求内容输入到该网站<https://www.ssldun.com/tools/csr-decoder.php#results>进行解码，如下图。

![img-w500](/images/201901211811.png) 

### CA签署请求
> 打开GoDaddy中所购买的SSL证书，填写相关内容，验证域名持有者信息，一般有两种方式，通过邮件或者在DNS添加解析记录，等待证书的签发。

### 证书的安装
> 待证书签发完成，找到证书下载页，选择对应的web服务类型，如果不存在，请选择其他，此处以nginx web服务类型为主

![img-w500](/images/201801211858.png) 

> 下载完成，将解压出的文件合并成一个文件，并命名为crt格式，即为证书文件  

```cat fd72a4fa7c1de0e3.crt gd_bundle-g2-g1.crt > glinux.top.crt```

![img-w500](/images/201901211912.png) 

> 更新配置文件以使用 SSL 证书
```
    server {
        listen       80 default_server;
        server_name  example.htrader.cn;
        return 301 https://$host$request_uri;
    }

    server {
        listen       443 ssl http2 default_server;
        server_name  example.glinux.top;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/nginx/conf.d/glinux.top.crt";
        ssl_certificate_key "/etc/nginx/conf.d/glinux.top.key";
        index index.php index.html index.htm
   }
```

## 参考文档  
+ 一篇文章让你搞懂 SSL 证书：<https://www.cnblogs.com/mafly/p/ssl.html>
+ SSL证书：<https://baike.baidu.com/item/SSL%E8%AF%81%E4%B9%A6/5201468?fr=aladdin>
+ 生成证书签名申请 (CSR) - Apache 2.x：<https://sg.godaddy.com/zh/help/csr-apache-2x-5269?v=1>
+ csr解码器：<https://www.ssldun.com/tools/csr-decoder.php#results>
+ CentOS 7 上的 NGINX： 安装证书：<https://sg.godaddy.com/zh/help/centos-7-nginx-27192>

