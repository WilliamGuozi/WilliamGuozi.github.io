# Nginx：Nginx limit_req limit_conn限速 

## 简介
+ Nginx是一个异步框架的Web服务器，也可以用作反向代理，负载均衡器和HTTP缓存，最常用的便是Web服务器。nginx对于预防一些攻击也是很有效的，例如CC攻击，爬虫，本文将介绍限制这些攻击的方法，可以使用nginx的ngx_http_limit_conn_module、ngx_http_limit_req_module这两个模块达到目的，该模块为nginx内置模块，yum安装即有，无需编译安装。本文就介绍nginx这两个模块的使用和细节，希望能够对需要的小伙伴有所帮助。

## 基本环境介绍
+ 两台机器，192.168.30.105和192.168.30.106均为 1c2g40g配置，106主机提供web服务，105主机部署ab工具。 

## web服务如下
![img-w500](/images/201811161439.png)  


## ab压测获取基础数据 

### 105 ab压测结果  
> 对web服务器index.html页面发送并发为1000总计1000000的请求测试，每个请求建立一个连接  
```ab -n 1000000 -c 1000  http://192.168.30.106:80/index.html```  


![img-w500](/images/201811201552.png)  
> 从测试结果来看，请求全部成功；有98%的请求在22ms以内就完成响应，有99%的请求在1007ms以内就完成响应，请求响应的最长时长为31077ms。



## nignx ngx_http_limit_conn_module模块
+ 该模块的功能是限制单个ip建立连接的个数。  

### 对nginx进行配置
```
http {
    limit_conn_zone $binary_remote_addr zone=one:10m;
    ...
    server {
        ...
        location / {
            limit_conn one 1;
        }    
```
> 限制每个ip连接的个数为一个

### 测试
> 对web服务器index.html页面发送并发为1000总计1000000的请求测试  
```ab -n 1000000 -c 1000  http://192.168.30.106:80/index.html``` 
![img-w500](/images/201811201614.png)  
> 从测试结果来看，请求全部成功；有98%的请求在58ms以内就完成响应，有99%的请求在1008ms以内就完成响应，请求响应的最长时长为31870ms。


### 测试效果
> 测试结果无变化，查众多文档，有问题，无答案，估计是个bug。


## nignx ngx_http_limit_req_module模块
+ 该模块的功能是限制单个ip请求的个数（请求频率）。  


### 对nginx进行配置

> 去掉之前limit_conn 配置，添加如下配置
```
http {
    limit_req_zone $binary_remote_addr zone=two:10m rate=1r/s;
    ...
    server {
        ...
        location / {
            limit_req zone=two;
        }    
```
> 限制请求的频率为单个ip每秒一个

### 测试
> 对web服务器index.html页面发送并发为1000总计1000000的请求测试  
```ab -n 1000000 -c 1000  http://192.168.30.106:80/index.html``` 
![img-w500](/images/201811201617.png)  
> 从测试结果来看，请求只有55个成功。


## 测试效果
> 有效的阻止了用户的请求。

## 测试过程web服务资源使用情况监控

> CPU利用  
![img-w500](/images/201811201710.png) 


> 网络接口流量  
![img-w500](/images/201811201701.png) 


> TCP连接数状态  
![img-w500](/images/201811201702.png) 

## 总结
+ 从测试的结果以及监控数据来看，limit_conn模块无效，不能起到任何限制作用；limit_req模块能够明显限制用户的请求内容，对于超出限制的请求，给予503的反馈；两者对服务器性能上都没有优化作用，拒绝的请求需要花费更多的硬件资源来处理，CPU消耗增多，接口流出的流量剧增。

## 参考文档

+ 官方文档：<http://nginx.org/en/docs>
+ 使用nginx limit_req限制用户请求速率：<https://www.centos.bz/2017/03/using-nginx-limit_req-limit-user-request-rate>
+ 关于limit_req和limit_conn的区别：<https://blog.csdn.net/u012566181/article/details/49968283>
+ ab压力测试报错：<https://www.cnblogs.com/felixzh/p/8295471.html>
+ ab性能测试结果分析：<https://www.cnblogs.com/gumuzi/p/5617232.html>
+ Rate Limiting with NGINX and NGINX Plus：<https://www.nginx.com/blog/rate-limiting-nginx/>