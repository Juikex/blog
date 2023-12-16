+++
title = '电子基建'
date = 2023-12-01T15:13:43+08:00
draft = true
tags = ["Linux","服务器","内网穿透"]
+++

记录一些平常用到的基础服务及配置，服务器是低功耗主机挂几块硬盘，系统Archlinux。公网ip买云服务器frp穿透
## Linux
国内最近的镜像站下载ArchlinuxISO烧录到U盘，主板固件不是很老的话都可以用UEFI启动。定制需求不是很大的话没必要按Archwiki的方法敲命令，使用
```
arch-install
```
解放双手，TUI界面选择自己需要的配置`insatll`即可。

## samba 
smb在各个平台均有支持，使用比较便捷。

安装`samba`软件包。
编辑`/etc/samba/samba.conf`
```
[Stuff]
   comment = comment some thing
   path = /location/to/share
   valid users = [User]
   public = no
   writable = yes
   create mask = 0765
```
也可以添加打印等参数。
之后添加samba用户
```
# smbpasswd -a samba_user

启用smb服务
systemctl enable smb --now
```
一个简易的smb现在已经可以使用了。

## Nginx部署
网页服务器是基础。Nginx相比于Apache更加轻量，对于个人服务而言也完全够用。
安装`nginx`包，并启动
```
# pacman -S nginx
# systemctl enable nginx --now
```
之后就可以在配置文件里添加`server`块(Apache中的vhost)
```
server {
    listen 79;
    listen [::]:79;
    server_name domainname0.tld;
    root /usr/share/nginx/domainname0.tld/html;
    location / {
        index index.php index.html index.htm;
    }
}
```
填入自己的域名及网站根目录即可，建议将网站配置文件放在单独的文件夹内方便管理如`/etc/nginx/sites`,之后在`/etc/nginx/nginx.conf`中添加引用
```
http {
    ...
    include sites-enabled/*;
}
```
然后打开`http://localhost`查看部署的网站

## Frp内网穿透
frp用途是内网穿透，将没有公网ip的服务暴露在公网上。分为两部分，服务器使用的Frps及客户端使用的Frpc。ArchlinuxCN源里已经打包了`frps`及`frpc`直接`pacman`安装即可。
### Frps
frp的主要配置都在服务端，编辑`/etc/frp/frps.ini`
```
[common]
bind_port = 7000
vhost_http_port = 80
vhost_https_port = 443
dashboard_port = 7500
dashboard_user = usr
dashboard_pwd = passwd
privilege_token = passswd
subdomain_host = example.com

log_file = /root/frps.log
log_level = info
log_max_days = 3
max_pool_count = 5
authentication_timeout = 900

tcp_mux = true
```
设置`privilage_token`作为使用凭证，手上有几个域名所以设置了`subdomain_host`用来设置二级域名。启动服务
```
# systemctl enable frps --now
```
防火墙开启端口之后就可以使用设置的用户密码登录ip:7500查看服务状态。

### Frpc
安装`frpc`,frps设定好以后就基本不需要动了，要添加穿透服务的话编辑客户端`/etc/frp/frpc.ini`:
```
[common]
server_addr = ipaddress
server_port = 7000
token = "passwd"

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port= 22
remote_port= 6000

[startpage]
type = http
local_ip = 127.0.0.1
local_port = 80
subdomain = startpage
```
`[common]`中填入ip及密码，其余为服务项，需要注意的是\[\]中的服务名称不能重复，即使是在不同客户端上。

实例中开启了ssh及http服务,ssh本地默认22端口映射远程6000端口登录使用
```
ssh user@ip -p 6000
```
http在frps中已经配置了一级域名`example.com`所以只要在frpc中填入二级域名即可，在正确配置服务器的情况下可以访问`startpage.example.com`，需要https的话可以使用`certbot`获取证书部署在服务器上，将类型改https端口改443即可，或者使用frps提供的https服务。

> frp的system service默认用户为nobody，系统中只有一个root用户的时候可能无法启动服务。

## Certbot配置https
现在网站已经可以在公网访问但是还没有证书,Certbot可以方便地安装和管理证书，服务端安装的是Nginx，所以安装`certbot`主体以及`cerbot-nginx`插件。运行：
```
sudo certbot --nginx
```
certbot会自动检测网站设置，在本地及域名配置正确的情况下certbot会列出部署的域名
```
Which names would you like to activate HTTPS for?
We recommend selecting either all domains, or all domains in a VirtualHost/server block.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: test.example.com
2: test2.example.com
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
```
数字选择需要添加证书的域名，2023/12的测试中不需要进行挑战了。之后certbot会更改网站的配置文件，添加443端口的访问同时屏蔽80端口：
```
server {
    server_name dom;
    location / {
       root /location;
       index index.php index.html index.htm;
    }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/dom/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/dom/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = dom) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    listen [::]:80;
    server_name dom;
    return 404; # managed by Certbot
```
在frpc中开启https服务就可以公网访问了。

## gitea配置

Gitea是一款支持自托管建立实例的版本控制平台，相比于gitlab等平台占用更小且功能丰富，除了标准的代码托管及审查功能还内置Action及webhooks，可玩性大大提高。
使用本地安装，PostgreSQL作为数据库，Nginx作为反向代理
```
sudo pacman -S gitea postgresql nginx

#切换到自动创建的postgres用户
su postgres
#初始化数据库
initdb -D /var/lib/postgres/data
systemctl enable postgresql --now
创建gitea数据库及用户
createuser -P gitea
createdb -O gitea gitea
```
因为服务器的数据库及gitea实例都是在一台机器上所以使用unix socket，编辑`/var/lib/postgres/data/pg_hba.conf`
```
local    gitea           gitea           peer
```
之后重启sql服务,启动gitea：
```
sudo systemctl restart postgresql
sudo systemctl enable gitea --now
```
之后在ip:3000端口上访问进行初始化设置，数据库选择`postgresql`，地址填`/run/postgresql/`,数据库用户名与密码根据自己设置填，其他根据需求填写。

需要注意的是仓库地址及LFS地址gitea需要有读写权限，否则会返回500只读文件系统。如果使用了自定义地址还需要`systemctl edit gitea`：
```
[Service]
ReadWritePaths=/etc/gitea/app.ini /path/to/gitea/repo
```
否则即使有读写权限创建仓库时也会失败，提示只读文件系统。
### 反向代理
gitea现在可以通过3000端口访问但是不太优雅，通过nginx反向代理在默认的80端口访问，需要Nginx添加配置：
```
server {
    listen 80;
    server_name ip;

    location /gitea/ {
        client_max_body_size 512M;
#        rewrite ^ $request_uri;
#        rewrite ^/gitea(/.*) $1 break;
#        proxy_pass http://127.0.0.1:3000$uri;
        proxy_pass http://127.0.0.1:3000/;
    }
}
```
注释段为gitea文档写法，使用rewrite去掉请求前面的gitea防止404，但是按照Nginx文档在Proxy_pass地址最后面加/，子路径前后都加/的话请求里的gitea是去掉的。

最后更改gitea配置文件加上`[server] ROOT_URL = http://127.0.0.1/gitea/`现在访问http://ip/gitea就可以使用gitea了。