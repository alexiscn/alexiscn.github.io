---
layout: post
title:  "play-with-vps"
date:   2016-05-12 15:48:10 +0800
categories: vps vpn
tag: 折腾人生
---

# 一、前言
我已经好久没有写博客了，最近受够了Linode的不稳定，打算换个VPS，平时写点小博客记录平时的点滴。于是选择了Vultr，并且Vultr有日本的节点，测试了一下速度还不错，比Linode快多了。本篇博客主要记录如何折腾VPS：

* A 搭建翻墙相关，shadowsocks + l2tp vpn
* B 使用Dropbox + Nginx + hexo 搭建自己的个人博客

![image](http://o6qskp3qg.bkt.clouddn.com/vultr_choose_instance.png?=598x311)

推荐大家使用vultr作为VPS，当然也为了自己的私信，戳下面的链接注册成功后大家都有好处 ：D
[http://www.vultr.com/?ref=6894070](http://www.vultr.com/?ref=6894070)

我创建是5刀/月的，地点：东京，系统：Ubuntu 12.04 x64

<!-- more -->

装完连上ssh，给系统装一些必要的软件，比如vim（默认安装了vi）、git

```
$ ssh root@your_vps_ip_address
# 输入VPS的密码，在vultr的面板中能找到
$ apt-get install vim
$ apt-get install git
```

# 二、搭建翻墙必备

### 1、手机VPN
选择安全性比较高的 l2tp的vpn，[直接戳这个链接](https://github.com/hwdsl2/setup-ipsec-vpn) 安装起来十分方便，一键安装，配置下账号就可以了。

在Ubuntu下执行如下的脚本就可以快速安装L2TP的VPN了，在执行脚本之前先编辑下脚本，指定账号密码以及秘钥。否则会使用随机账号密码。

```
wget https://git.io/vpnsetup -O vpnsetup.sh
vim vpnsetup.sh
[Edit and replace IPSEC_PSK, VPN_USER and VPN_PASSWORD with your own values]
sudo sh vpnsetup.sh
```

大约3-5分钟左右，命令全部执行好后，会输出你的VPN配置信息。

```
IPsec/L2TP VPN server setup is complete!
Connect to your new VPN with these details:
Server IP: <your_server_ip>
IPsec PSK: <your_psk>
Username: <your_username>
Password: <your_password>

Write these down. You'll need them to connect!

Important Notes:   https://git.io/vpnnotes
Setup VPN Clients: https://git.io/vpnclients

```

如果想要添加更多的账号，可以编辑 `/etc/ppp/chap-secrets` 文件，按照格式添加账号密码即可。

```
root@vultr:/etc/ppp# cat chap-secrets
# Secrets for authentication using CHAP
# client  server  secret  IP addresses
"tom" l2tpd "tompassword" *
"jack" l2tpd "jackpassword" *
```

注意：重启Server后，VPN的服务也会自动启动，为我们解决了重启Server后再次启动VPN服务的后顾之忧。

### 2、路由器Shadowsocks

![image](http://o6qskp3qg.bkt.clouddn.com/vultr-shadowsocks.png?=600x)

家里买了一个华硕AC66U，刷了一个[梅林固件](https://www.chiphell.com/thread-1243565-1-1.html)，挂SS上然后在家里wifi上网就自由翻墙了，这也是我买VPS的一个重要原因。怎么样刷梅林固件我就不多说， 教程上都有。

安装SS也十分简单，可以按照[Shadowsocks官网中的文档安装](https://github.com/shadowsocks/shadowsocks/wiki)

```
$ apt-get install python-pip
$ pip install shadowsocks
```

通常我们选择创建配置文件来启动Shadowsocks，创建`/etc/shadowsocks.json`文件，配置信息如下：

```
$ vim /etc/shadowsocks.json
[input following]
{
"server":"my_server_ip",
"server_port":8388,
"local_address": "127.0.0.1",
"local_port":1080,
"password":"mypassword",
"timeout":300,
"method":"aes-256-cfb",
"fast_open": false
}
```

ok，接下来运行一下下面的命令就可以让ssserver在后台运行了。

```
$ ssserver -c /etc/shadowsocks.json -d start
```

**注意**：由于第一步中安装VPN后，该脚本会创建并修改防火墙的配置，所以默认8388端口被禁用了，我们需要开启8388端口，当然还有80端口。编辑`/etc/iptables.rules`，在`-A INPUT -p tcp --dport 22 -j ACCEPT`下面增加开启8388以及80端口的代码，然后重启下iptables：`/etc/iptables restart`

```
-A INPUT -p tcp --dport 22 -j ACCEPT
-A INPUT -p tcp --dport 8388 -j ACCEPT
-A INPUT -p tcp --dport 80 -j ACCEPT
```


### 3、设置Shadowsocks开机自动启动

设置开机就启动Shadowsocks，这个折腾了我比较久的时间，相对也较为复杂，所以单独出来一小节...网上的教程是直接编进 `/etc/rc.local`文件，增加一句
`ssserver -c /etc/shadowsocks.json -d start`，但实际上折腾许久都没有效果，于是换了一种方法。

创建 `/etc/init.d/shadowsocks` 文件，复制如下脚本：

```
#!/bin/sh

start(){
ssserver -c /etc/shadowsocks.json -d start
}

stop(){
ssserver -c /etc/shadowsocks.json -d stop
}

case "$1" in
start)
start
;;
stop)
stop
;;
reload)
stop
start
;;
*)
echo "Usage: $0 {start|reload|stop}"
exit 1
;;
esac
```

然后再创建 `/etc/init/shadowsocks.conf` 文件，复制如下脚本：

```
start on (runlevel [2345])
stop on (runlevel [016])
pre-start script
/etc/init.d/shadowsocks start
end script
post-stop script
/etc/init.d/shadowsocks stop
end script
```

然后更新到开机启动中

```
$ sudo update-rc.d shadowsocks defaults
```

现在重启你的server，shadowsocks服务会自动启动。


#三、搭建个人博客

### 1、安装Dropbox
由于博客是公开的，建议你新注册一个Dropbox的账号来存放博客文件，不要使用自己的主账号。下载Dropbox在linux下的安装包，解压到当前的目录，执行dropboxd命令安装Dropbox

```
$ wget -O dropbox_lnx "https://www.dropbox.com/download?plat=lnx.x86_64"
$ tar -xzf dropbox_lnx
$ ~/.dropbox-dist/dropboxd
```

在Console上会一直刷一个网址，提示给登录你的Dropbox账号，你可以在你本机浏览器里面输入这个网址并且登录你的Dropbox账号，登录成功后会提示你关联成功，此时你输入`Ctrl+C`就可以了.
这时候你会发现在你的Root目录下会有一个Dropbox目录。Dropbox 提供一个python脚本用于管理linux上的Dropbox,Dropbox也提供这个脚本的[使用文档](https://www.dropbox.com/help/9192)。

```
$ wget https://linux.dropbox.com/packages/dropbox.py
$ chmod +x ./dropbox.py
./dropbox.py autostart //加入到自动启动中
./dropbox.py start // 开始同步
./dropbox.py status //查看当前的同步状态
```
ubuntu上更多使用dropbox的请戳 [Dropbox](https://help.ubuntu.com/community/Dropbox)

### 2、安装Hexo
安装[Hexo](https://hexo.io/)相对来说十分简单，但是需要先安装nodejs以及npm

```
$ apt-get install nodejs
$ apt-get install nodejs-legacy
$ apt-get install npm
$ npm install -g hexo-cli
```

在Dropbox目录下搭建自己的博客

```
$ cd /Dropbox/
$ hexo hexo
```

会在Dropbox目录下创建

### 3、安装Nginx

Hexo是静态博客，需要Ngnix配合着使用

```
$ apt-get install nginx
```

注意：安装完nginx是为用户nginx，可能没有一些文件夹的权限，你可以给用户nginx增加目录权限或者简单的将nginx的user修改为root。配置好的nginx.conf 大概如下所示：

```
user root;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
worker_connections 1024;
}

http {
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
'$status $body_bytes_sent "$http_referer" '
'"$http_user_agent" "$http_x_forwarded_for"';

access_log  /var/log/nginx/access.log  main;

sendfile            on;
tcp_nopush          on;
tcp_nodelay         on;
keepalive_timeout   65;
types_hash_max_size 2048;

include             /etc/nginx/mime.types;
default_type        application/octet-stream;

# Load modular configuration files from the /etc/nginx/conf.d directory.
# See http://nginx.org/en/docs/ngx_core_module.html#include
# for more information.
include /etc/nginx/conf.d/*.conf;

server {
listen       80 default_server;
listen       [::]:80 default_server;
server_name  shuifeng.me;
root         /usr/share/nginx/html;

# Load configuration files for the default server block.
include /etc/nginx/default.d/*.conf;

location / {
root /root/Dropbox/hexo/public;
index index.php index.html index.htm;
}

error_page 404 /404.html;
location = /40x.html {
}

error_page 500 502 503 504 /50x.html;
location = /50x.html {
}
}
}
```


配置好nginx后，可以使用 `nginx -t`测试下nginx是否可以用，如果遇到nginx: [emerg] a duplicate default server for 0.0.0.0:80 in /etc/nginx/nginx.conf:75 的错误，删除/etc/nginx/sites-available/default文件，重新启动服务即可

安装Incron用于同步更新，即你在本机写了篇博客，上传到了Dropbox后，VPS上的Dropbox下载到新的文章，自动部署。

```
$ apt-get install incron
```

，在最下面增加如下代码：

```
/root/Dropbox/hexo/source/_posts/ IN_MOVE,IN_MODIFY,IN_CREATE,IN_DELETE /root/runhexo.sh
```

这个配置的意思是当hexo中posts文件夹发生更改，如移动、修改、创建、删除的时候，执行runhexo.sh命令，那么runhexo.sh实际上就是生成hexo的静态文件，脚本如下：

```
#!/usr/bin/env bash
exec 200<$0
flock -n 200 || exit 1
sleep 10
cd /root/Dropbox/hexo && hexo clean && hexo generate
```

incron 并不能监听hexo主题变化..所以如果更改了主题，需要自己手动生成。

写博客


安装主题[Next](https://github.com/iissnan/hexo-theme-next)
11

### 4、博客常见操作

好难用

在markdown中插入图片

```
![image](http://o6qskp3qg.bkt.clouddn.com/vultr_choose_instance.png)
```

我们还可以指定图片的尺寸，可以同时指定宽高，也可以只指定宽，让高度自适应

```
![image](http://o6qskp3qg.bkt.clouddn.com/vultr_choose_instance.png=598x311)
![image](http://o6qskp3qg.bkt.clouddn.com/vultr_choose_instance.png=598)
```