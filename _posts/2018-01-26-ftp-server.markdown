---
layout: post
title:  "VPS上增加FTP服务"
date:   2018-01-26 10:43:10 +0800
categories: VPS SFTP
tag: 折腾人生
---

VPS的另一个玩法就是当做一个FTP服务，我们可以传一些文件（比如HTML）,Vultr上开启FTP十分简单，自带了SFTP，无需安装 vsftpd。


1. 首先，创建sftpusers组

```sh
$ sudo groupadd sftpusers
$ sudo sed -i "s/Subsystem sftp \/usr\/lib\/openssh\/sftp-server/#Subsystem sftp \/usr\/lib\/openssh\/sftp-server/" /etc/ssh/sshd_config
```

2. 编辑 `/etc/ssh/ssh_config`，增加如下的配置

```sh
#enable sftp
Subsystem sftp internal-sftp

Match Group sftpusers
   ChrootDirectory %h #set the home directory
   ForceCommand internal-sftp
   X11Forwarding no
   AllowTCPForwarding no
   PasswordAuthentication yes

```

3、编辑 `~/.bash_profile`，复制下面的代码，然后`source ~/.bash_profile` 使其生效。代码的意思是创建sftp的用户，并且使其没有login的权限。同时在用户目录下创建files目录，用来存放ftp的文件，并且给文件夹增加相应的权限。做成函数可以方便配置多个ftpuser。

```bash
# usage: create_sftp_user <username>
function create_sftp_user() {
    # create user
    sudo adduser $1

    # prevent ssh login & assign SFTP group
    sudo usermod -g sftpusers $1
    sudo usermod -s /bin/nologin $1

    # chroot user (so they only see their directory after login)
    sudo chown root:$1 /home/$1
    sudo chmod 755 /home/$1

    sudo mkdir /home/$1/files
    sudo chown $1:$1 /home/$1/files
    sudo chmod 755 /home/$1/files
}
```

4、创建ftp 用户user1

```sh
$ create_sftp_user user1
```

5、配置Nginx etc/nginx/nginx.conf

将 `user www_data;` 修改为 `user root;`

编辑 `/etc/nginx/sites-enabled/default`

```
server {
    ...

    location /ftp {
        alias /home/user1/files;
		autoindex on;
    }
}
```


最后用FileZilla 连接，记得协议选择 SFTP

参考文档 [Setup SFTP-Only User Accounts On Ubuntu 14](https://www.vultr.com/docs/setup-sftp-only-user-accounts-on-ubuntu-14)