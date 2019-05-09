---
layout: post
title:  "用Jekyll搭建博客"
date:   2018-01-24 16:28:10 +0800
categories: Jekyll VPS Nginx Hooks
---

更新：现在直接用的github pages. 更加方便

发现基于nodejs的[Hexo](https://hexo.io/zh-cn/index.html) 比较占磁盘空间，导致VPS的硬盘基本满了（15G），再加上一些奇怪的环境错误导致apt-get install 出错，于是打算重新折腾下VPS，科学上网就不说了，重点说一下折腾比较耗时的博客搭建。

## 环境配置

> 
> Vultr 东京节点
> Ubuntu 14.04
> Client： Mac

## 1、ssh到服务器安装必要的软件

```bash
laptop$ ssh root@your_server_ip
server$ apt-get install vim
server$ apt-get install git
```

由于 Ubuntu自带的Ruby版本较低，我们用RVM来管理安装Ruby

```bash
# 安装 rvm
server$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
server$ curl -sSL https://get.rvm.io | bash -s stable
server$ source ~/.bashrc
server$ source ~/.bash_profile
# 检查rvm是否安装成功
server$ rvm -v 
```

有时候安装完rvm，会提示你做一些必要的步骤来使rvm生效，按照提示的步骤做就行了。安装ruby2.3.0

```bash
server$ rvm install 2.3.0
server$ gem install jekyll bundler
```

博客所需要的软件基本搭建好了。为了安全寄

## 2、添加git用户，用于远程push仓库

新建git用户，记住密码

```bash
server$ sudo adduser git
```

为了安全起见，只能让git用户通过ssh访问git，而不应该让git用户拥有shell的权限。需要禁用git的shell权限，编辑`/etc/passwd`文件，找到如下的脚本

```bash
git:x:1001:1001:,,,:/home/git:/bin/bash
```

替换为

```bash
git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
```

这样git用户只有git-shell的权限了。


## 3、创建ssh key

我们需要创建，这样就不需要每次push的时候都需要git用户的密码

```bash
laptop$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]
Enter passphrase (empty for no passphrase): [Type a passphrase]
```

添加到本机

```bash
laptop$ eval "$(ssh-agent -s)"
Agent pid 59566
```

```bash
root@server$ cd ~
root@server$ mkdir .ssh && cd .ssh
root@server$ touch authorized_keys
root@server$ vim authorized_keys
```

https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/

## 3、创建git 目录


我把git放在 `/var/repo/blog.git` 下，然后创建一个裸仓库

```bash
root@server$ mkdir /var/repo
root@server$ cd /var/repo
root@server$ mkdir blog.git
root@server$ cd blog.git
root@server$ git init --bare
```

用于存放Jekyll生成的静态文件在 `/var/www/blog`下。需要给 blog.git blog 目录加上git用户的权限。


### 4、配置hooks

hooks的脚本跟官方文档略有不同，增加了`#!/bin/sh`，指定具体的jekyll命令路径

```bash
#!/bin/sh
GIT_REPO=/var/repo/blog.git
TMP_GIT_CLONE=/var/tmp/blog
PUBLIC_WWW=/var/www/blog

git clone $GIT_REPO $TMP_GIT_CLONE
/usr/local/rvm/gems/ruby-2.2.7/wrappers/jekyll build -s $TMP_GIT_CLONE -d $PUBLIC_WWW
#rm -Rf $TMP_GIT_CLONE
```

稍微解释一下hooks脚本的内容

最后需要给post-receive增加可执行权限。

```bash
root@server$ chmod +x hooks/post-receive
```


### 5、写博客

正常写博客，然后提交的时候运行如下的命令就可以了

```bash
laptop$ git add *
laptop$ git commit -m 'new blog'
laptop$ git push blog master
```


# 配置 HTTPS

我选用Let's Encrypt来给博客增加 HTTPS证书，用certbot-auto管理HTTPS证书，首先下载certbot-auto

```bash
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
./certbot-auto certonly --agree-tos --webroot -w /var/www/blog -d shuifeng.me
```

生成证书路径：/etc/letsencrypt/live/example.com

配置 Nginx，让证书生效

编辑vim /etc/nginx/sites-enabled/default文件，打开 HTTPS server 下面的server的注释，然后替换之前得到的key的路径到 ssl_certificate、ssl_certificate_key

```
# HTTPS server

server {
	listen 443;
	server_name localhost;

	#root html;
        root /var/www/blog;
	index index.html index.htm;

	ssl on;
	ssl_certificate /etc/letsencrypt/live/shuifeng.me/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/shuifeng.me/privkey.pem;

	ssl_session_timeout 5m;

	ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
	ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
	ssl_prefer_server_ciphers on;

	location / {
		try_files $uri $uri/ =404;
	}
}
```

重新加载nginx 使之生效。

```sh
$ /etc/init.d/nginx reload
```


免费的 Let's Encrypt 有效期是90天，如果超过90天，我们没有自动续的话，HTTPS就过期了，所以我们需要自动Renew证书，也很简单

```sh
$ crontab -e
```

然后输入如下的命令，每个月1号凌晨3点都会自动执行renew，同时重启nginx

```
0 3 1 * * /root/certbot-auto renew –renew-hook "/etc/init.d/nginx reload"
```


# 常见问题


## 1、jekyll not found

这是安装官方文档配置出现的问题，在上面配置hooks的时候已经修正过来了，就是给全 Jekyll执行路径


## 2、Invalid US-ASCII character "\xE2" on line 10

```bash
Conversion error: Jekyll::Converters::Scss encountered an error while converting 'css/main.scss':
Invalid US-ASCII character "\xE2" on line 10
```

试了很多方法，发现需要改 sass 中的 `engine.rb` 源码

```bash
$ which ruby
$ cd /usr/local/rvm/gems/ruby-2.x.x/gems
$ ls -l
```

找到最新版本的sass，比如sass-3.5.5

```bash
$ cd /usr/local/rvm/gems/ruby-2.x.x/gems/sass-3.5.5/lib/sass
$ vim engine.rb
```


在require的结尾处增加如下的代码

```bash
Encoding.default_external = Encoding.find('utf-8')
```

## 3、Enter passphrase for key '/Users/your_name/.ssh/id_rsa': 

将如下的命令添加到本机的 ~/.bash_profile中

```bash
eval `ssh-agent` 
ssh-add
```

然后source使其生效会提示你输入 passphrase

```bash
laptop$ source ~/.bash_profile
```


## 4、control characters are not allowed at line 1 column 1

push到远程服务端，生成静态页面的时候可能会提示上述错误信息。因为markdown中有一些以外的符号

![](/assets/images/jekyll_error_control_characters.jpg)

用`SublimeText`打开markdown文件就能看到，删掉就可以了