---
layout: post
title: "Docker 使用socket 代理"
tags: DevOps Linux
---

## 简介
----
* 小伙伴们好，今天介绍一个docker镜像拉取失败的解决办法，也就是使用7层代理工具shadowsock的方式
* 目前对于镜像拉去失败，主流的解决方案，要么国内外做镜像的同步， 要么就是使用代理; 而做镜像同步会耗费较多的资源，不是最优的选择， 建议使用代理的方式
* 据我所知，目前存在的拉取失败，要么是国内服务器拉取国外镜像仓库镜像失败，比如 docker.io、us.gcr.io；或国外服务器拉取国内镜像仓库镜像失败，比如 registry.cn-hangzhou.aliyuncs.com
* 当然我们可以通过 在能够拉取的地方下载镜像命令 `docker save -o myimages.gz williamguozi/httpd:v0.1 williamguozi/httpd:v0.2`，之后在该服务器上加载该镜像 `docker load -i myimages.gz`, 偶尔一两次还可以，持续更新就很难接受了
* 今天就将该代理方式介绍给大家，希望对需要的小伙伴有所帮助
* 逻辑结构如图

![img-w500](/images/202006081723.png)

<font color="#dd0000">所有操作在需要拉取镜像的服务器上执行</font>
## 依赖环境
* 该依赖是shadowsock运行环境所需要的库

```bash
apt update
apt-get install build-essential wget -y
wget https://github.com/jedisct1/libsodium/releases/download/1.0.10/libsodium-1.0.10.tar.gz
tar xzvf libsodium-1.0.10.tar.gz
cd libsodium*
./configure
make -j8 && make install
echo /usr/local/lib > /etc/ld.so.conf.d/usr_local_lib.conf
ldconfig
```


## ss 客户端
* 安装 `shadowsock` 客户端，用于连接shadowsock服务器
* `shadowsock` 服务器自行搜索资料建设，选址需满足拉取镜像的服务器可达，并且可访问目标镜像源
* 以下为客户端参考配置，实际情况应更改 server（可只填写一个），server_port，password，method

```bash
apt-get install python-pip -y
pip install shadowsocks

cat >  /etc/shadowsocks.json << EOF
{
"server":["100.100.100.100","200.200.200.220"],
"server_port":2911,
"local_port":1080,
"password":"password",
"timeout":600,
"method":"chacha20"
}
EOF

#启动服务
sslocal -c /etc/shadowsocks.json -d start
```


## 安装privoxy 转socket 到http https工具
* shdowsock 代理后的数据为socket格式，需要转换为http和https

```bash
apt-get install privoxy -y

# 需添加如下配置
vim /etc/privoxy/config
        forward-socks5   /               127.0.0.1:1080  .
listen-address 127.0.0.1:8118

systemctl start privoxy
systemctl enable privoxy
```

## docker 使用代理
* 首先需检测docker重启是否会导致容器重启
* <font color="#dd0000">该配置文件需要有如下配置，否则重启docker服务会导致所有容器重启</font>
* <font color="#dd0000">如果之前没有该配置，添加后，第一次重启仍然会导致所有容器重启</font>

```bash
cat > /etc/docker/daemon.json << EOF
{
  "live-restore": true,
  "group": "docker"
}
EOF

cat docker.service |grep process
# kill only the docker process, not all processes in the cgroup
KillMode=process
```

* 修改容器服务启动参数
* `Environment=HTTP_PROXY=http://127.0.0.1:8118/` `Environment=HTTPS_PROXY=http://127.0.0.1:8118/` 代理http https
* `Environment=NO_PROXY=localhost,127.0.0.1,docker.io` 不要代理的镜像仓库源域名，否则将全部代理

```bash
vim /etc/systemd/system/multi-user.target.wants/docker.service
[Service]
Environment=HTTP_PROXY=http://127.0.0.1:8118/
Environment=HTTPS_PROXY=http://127.0.0.1:8118/
Environment=NO_PROXY=localhost,127.0.0.1,docker.io

systemctl daemon-reload
systemctl restart docker
```

## 总结
* 在该服务器上重新拉取阿里云镜像，发现可以正常拉取
* 如有疑问或更好的解决方案，欢迎留言交流


## 参考文档
* docker 使用 socks5代理：<https://blog.csdn.net/S1234567_89/article/details/73223200>
* 安装libsodium库解决libsodium not found问题: <https://www.debugnode.com/ubuntul_ibsodium>
* 如何保证 docker daemon重启，但容器不重启: <https://blog.csdn.net/qianggezhishen/article/details/71082689>