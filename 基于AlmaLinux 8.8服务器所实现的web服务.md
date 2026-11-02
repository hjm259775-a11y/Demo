# 基于AlmaLinux 8.8服务器所实现的web服务（WordPress）



**命令补全**

刚买的服务器没有这项功能，我们需要进行下载

```
[root@localhost ~]# dnf install -y bash-completion
[root@localhost ~]# source /etc/profile.d/bash_completion.sh
```





## 安全防护

首先先利用SSH输入IP地址来连接你的服务器，同时，新买的服务器要**先构筑防御措施**

```
[root@localhost ~]# firewall-cmd --permanent --add-service=http

[root@localhost ~]# firewall-cmd --permanent --add-service=https

[root@localhost ~]# firewall-cmd --reload

[root@localhost ~]# firewall-cmd --list-services
cockpit dhcpv6-client http https ssh

```

好，安全服务基本完成



## 软件下载



我们需要这四个软件

*   nginx
*   php83-php-fpm
*   php83-php-mysqlnd
*   mariadb-server

这些是让动态网站能跑起来的软件



**下载软件前需要更新软件源**



首先明确一点，我们需要PHP8.3，Remi 商店里有PHP 8.3，但是Remi 里有些包依赖 EPEL 里的东西，不先装 EPEL，Remi 可能装不全。所以**需要先装EPEL，再装Remi**

```
[root@localhost ~]# dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

[root@localhost ~]# dnf install -y https://rpms.remirepo.net/enterprise/remi-release-8.rpm                   
```



如果在安装时报错，可能是AlmaLinux 系统自带的 GPG 密钥过期/不匹配，导致它在尝试升级 almalinux-release这个基础包时验签失败，整个安装事务被拦下了。

```
[root@localhost ~]# rpm --import https://repo.almalinux.org/almalinux/RPM-GPG-KEY-AlmaLinux
# 从 AlmaLinux 官方下载最新的 GPG 密钥并导入。这是解决 GPG check FAILED 的标准做法，多个来源都确认有效。

[root@localhost ~]# dnf clean packages
# 清掉刚才下载失败残留的缓存包
```

敲出这些命令以解决，然后重新下载就行了





**然后我们下载那四个软件**

```
[root@localhost ~]# dnf install -y nginx mariadb-server php83-php-fpm php83-php-mysqlnd \
  php83-php-gd php83-php-mbstring php83-php-xml php83-php-zip \
  php83-php-intl php83-php-bcmath php83-php-curl php83-php-opcache
```

具体作用是这些：

-   `nginx`、`mariadb-server`、`php83-php-fpm`、`php83-php-mysqlnd`：四个核心，必须有。
-   `php83-php-gd`：缩略图、图片处理。
-   `php83-php-mbstring`：中文、emoji 不乱码。
-   `php83-php-xml`：RSS、站点地图。
-   `php83-php-zip`：后台装主题/插件解压。
-   `php83-php-intl`：多语言。
-   `php83-php-bcmath`：金额等高精度计算。
-   `php83-php-curl`：WordPress 更新、调外部 API。
-   `php83-php-opcache`：缓存，提速。



**开启服务**

```
[root@localhost ~]# systemctl enable --now nginx mariadb php83-php-fpm

[root@localhost ~]# ls /var/opt/remi/php83/run/php-fpm/www.sock
/var/opt/remi/php83/run/php-fpm/www.sock
# PHP-FPM 启动后会在 /run/php-fpm/ 下创建 socket 文件（通常叫 www.sock）。Nginx 的 fastcgi_pass 必须指向这个文件，不过很显然，PHP83版本本文件的路径不太一样
```





## 数据库





动态网站的内容存在数据库里。WordPress 运行时不断从数据库读数据、往数据库写数据，所以要先建好库和用户，给 WordPress 一个能连库存数据的地方。



**初始化数据库：**

```
[root@localhost ~]# mysql_secure_installation
# 对 MariaDB 做安全初始化：设 root 密码、删匿名用户、禁止 root 远程登录、删 test 库

[root@localhost ~]# mysql -u root -p
# 用刚设置的root账户进入 MariaDB

MariaDB [(none)]> CREATE DATABASE wp DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
建一个叫 wp 的库。WordPress 的文章、用户、评论、设置以后都存这里。

MariaDB [(none)]> CREATE USER 'wpuser'@'localhost' IDENTIFIED BY '密码';
建数据库用户 wpuser并且设密码。WordPress 用这个身份连库，不用 root。

MariaDB [(none)]> GRANT ALL PRIVILEGES ON wp.* TO 'wpuser'@'localhost';
把 wp 库的全部权限授予 wpuser。当然，也仅此wp库了

MariaDB [(none)]> FLUSH PRIVILEGES;
刷新权限表，让授权立即生效
```



>   注意：库名 wp、用户名 wpuser、wpuser 的密码。这三样要记住，安装 WordPress 时要填。



| 名字     | 是什么           | 权限               |
| :------- | :--------------- | :----------------- |
| `root`   | MariaDB 总管理员 | 所有库，全部权限   |
| `wp`     | 一个库           | 存 WordPress 数据  |
| `wpuser` | 一个数据库用户   | 只有 `wp` 库的权限 |



## 下载 WordPress 代码，放到网站目录





```
[root@localhost ~]# cd /tmp/
[root@localhost tmp]# curl -O https://wordpress.org/latest.tar.gz ————————————下载WordPress的压缩包

[root@localhost tmp]# tar -xf latest.tar.gz -C /var/www ————————————————————解压到/var/www

[root@localhost tmp]# chown -R nginx:nginx /var/www/wordpress/ ——————————————递归修改文件所属者和组，让nginx可以自己用


```





## 配 Nginx，把 PHP 请求转给 PHP-FPM





-   `$uri`：当前请求的路径
-   `$args`：URL 里 `?` 后面的参数
-   `$document_root`：当前 `root` 指令的值
-   `$fastcgi_script_name`：要执行的脚本路径

这些名字也是 Nginx 定死的，拼错就取不到值。

------

为什么是这几个参数、这个顺序

`try_files` 的语法是：

```
try_files 文件1 文件2 ... 最后兜底;
```



-   前面可以列任意多个“尝试项”，Nginx 按顺序找。
-   **最后一个必须是兜底**（内部重定向或返回码），不能也是文件。

所以 `try_files $uri $uri/ /index.php?$args;` 的意思是：

1.  `$uri` → 先试真实文件
2.  `$uri/` → 再试目录
3.  `/index.php?$args` → 最后兜底，转给 index.php





```
[root@localhost tmp]# vim /etc/nginx/conf.d/wordpress.conf
————————————————————————————————————————————————————————————————————————————————
server{
        listen 80;
        server_name IP;
        root /var/www/wordpress;
        index index.php index.html;
        client_max_body_size 64M; ——————————————————Nginx 允许的最大上传大小 64M。默认 1M，传主题/图片会被拦，所以调大。
        location / { ———————————————————————————————用户访问的网址不以 .php 结尾时，走这里
           try_files $uri $uri/ /index.php?$args;
        }
        location ~ \.php$ { ————————————————————————用户访问的网址以 .php 结尾时，走这里
                include fastcgi_params;
                fastcgi_pass unix:/var/opt/remi/php83/run/php-fpm/www.sock;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
        location ~ /\.(?!well-known) { ——————————————用户访问以 /. 开头的网址时，走这里。比如 /.git/config、/.env。deny all 直接拒绝，返回 403。这些是隐藏文件，可能含密码或源码信息，不能让人下载
                deny all;
        }
}
————————————————————————————————————————————————————————————————————————————————

```



修改配置文件





```shell
[root@localhost ~]# vim /etc/opt/remi/php83/php-fpm.d/www.conf ————————————————修改php的配置文件
———————————————————————————————————————————————————————————————————————————————— 
 24 user = nginx

 26 group = nginx

 48 listen.owner = nginx
 49 listen.group = nginx
 50 listen.mode = 0660
———————————————————————————————————————————————————————————————————————————————— 

[root@localhost ~]# ls -l /var/opt/remi/php83/run/php-fpm/www.sock —————可以看到文件所有者和组是root，nginx打不开
srw-rw----+ 1 root root 0 Oct 27 09:15 /var/opt/remi/php83/run/php-fpm/www.sock
[root@localhost ~]# chown nginx:nginx /var/opt/remi/php83/run/php-fpm/www.sock ——————————临时修改为nginx
[root@localhost ~]# ls -l /var/opt/remi/php83/run/php-fpm/www.sock
srw-rw----+ 1 nginx nginx 0 Oct 27 09:15 /var/opt/remi/php83/run/php-fpm/www.sock
[root@localhost ~]# mkdir -p /etc/systemd/system/php83-php-fpm.service.d/ ——————————让 systemd 以后每次重启 PHP-FPM 时，都自动把权限给 nginx
[root@localhost ~]# echo -e "[Service]\nRuntimeDirectoryMode=0750\nExecStartPost=/bin/chown nginx:nginx /var/opt/remi/php83/run/php-fpm/www.sock" > /etc/systemd/system/php83-php-fpm.service.d/override.conf ————写一个配置让 systemd 生效
[root@localhost ~]# systemctl daemon-reload ——————————————————————————————————重新加载 systemd 配置并重启 PHP-FPM
[root@localhost ~]# systemctl start php83-php-fpm
[root@localhost ~]# ls -l /var/opt/remi/php83/run/php-fpm/www.sock
srw-rw----+ 1 nginx nginx 0 Oct 27 09:24 /var/opt/remi/php83/run/php-fpm/www.sock
[root@localhost ~]# systemctl restart nginx

```





## WordPress 安装向导



用浏览器输入你的IP地址，然后一步一步跟着引导走，最后登录



如果以后想在其他电脑上登录可以在IP地址后加上/wp-admin

![image-20261028140049067](https://raw.githubusercontent.com/hjm259775-a11y/image-hosting/main/test/20261028140056994.png)



当你看到这个界面，就代表装好了



## 连接域名



默认情况下，我们是通过服务器 IP访问网站的。但 IP 难记、不安全（浏览器提示“不安全”），且无法申请免费证书。

连接域名可以实现：

1.  用一个好记的名字访问网站。
2.  隐藏服务器真实 IP，防止被攻击。
3.  免费获得 HTTPS 加密



在创建好域名和你URL的对应关系之后，修改/etc/nginx/conf.d/wordpress.conf配置文件

将原本的IP换成你的域名，并且修改你数据库中的网址

```
mysql -u root -p
USE wp;
UPDATE wp_options SET option_value = 'http://网址' WHERE option_name IN ('siteurl', 'home');
EXIT;
```



再对/wp-admin进行访问，就会发现登录界面又出来了











## https建立



先在服务器上安装免费 SSL 证书

服务器是 AlmaLinux 8，可以用 Certbot 从 Let's Encrypt 申请免费证书，它会自动帮你改好 Nginx 配置。

**1. 安装 Certbot 和 Nginx 插件**
在服务器终端运行：

```
dnf install -y epel-release
dnf install -y certbot python3-certbot-nginx
```

**2. 申请并自动配置证书**

```
certbot --nginx -d 域名
```

过程中它会问你邮箱、同意协议，然后自动把证书装好并修改你的 Nginx 配置文件

**3. 确认证书装好了**

```
ls /etc/letsencrypt/live/域名/
```





**修改 Nginx 配置，确保 PHP 正常**

Certbot 会自动改你的 `wordpress.conf`，之后需要手动在 `location ~ \.php$` 块里加一行，告诉 PHP 现在确实是 HTTPS：



```
vim /etc/nginx/conf.d/wordpress.conf
————————————————————————————————————————————————————————————
server{
        server_name 域名;
        root /var/www/wordpress;
        index index.php index.html;
        client_max_body_size 64M;
        location / {
           try_files $uri $uri/ /index.php?$args;
        }
        location ~ \.php$ {
                include fastcgi_params;
                fastcgi_pass unix:/var/opt/remi/php83/run/php-fpm/www.sock;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param HTTPS on; ——————————————————————————————————————————————————————————添加这项
        }
        location ~ /\.(?!well-known) {
                deny all;
        }
————————————————————————————————————————————————————————————
```



加完后测试并重载：

```
nginx -t && systemctl reload nginx
```



之后把 Cloudflare 改成“完全（严格）”













