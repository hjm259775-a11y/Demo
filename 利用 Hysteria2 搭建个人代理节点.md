# 利用 Hysteria2 搭建个人代理节点



## 检查服务器配置



*   先**检查下自身服务器的配置情况**，本次个人节点选用`UDP/443`端口，需要确保本端口没有被占用，以及保证硬盘空间充裕

```
[root@localhost ~]# ss -lntup | grep -E ':(80|443) '
[root@localhost ~]# nproc; free -h | head -2; df -h / | tail -1
```



## 下载安装 Hysteria 2



*   **静默下载` Hysteria 2 `官方的安装脚本**

```
[root@localhost ~]# curl -fsSL https://get.hy2.sh/ -o /tmp/hy2-install.sh
```

`-f `如果服务器返回错误（比如 404 找不到文件），直接报错退出，不下载错误代码。
`-s` 静默模式，不显示进度条和乱七八糟的信息。
`-S` 配合 -s 用，虽然静默，但如果报错了，要把错误信息显示出来。
`-L` 如果下载地址发生了重定向，自动跟着跳转去下载最终的文件。



*   **运行脚本进行安装**

```
[root@localhost ~]# bash /tmp/hy2-install.sh

[root@localhost ~]# hysteria version
```

安装后查看下版本信息，是否正确安装



## 修改配置文件





```
[root@localhost ~]# vim /etc/hysteria/config.yaml
————————————————————————————————————————————————————————————————————————————
listen: :443 ——————————————————————————————去掉注释符号

# acme:
#   domains:
#     - your.domain.net
#   email: your@email.com —————————————————这里是指让他自己申请证书，注释掉，我们选用自己申请的证书

tls:
  cert: /etc/hysteria/fullchain.pem
  key: /etc/hysteria/privkey.pem ——————————使用已有证书这么写就行，一会儿把证书复制到这个路径

auth:
  type: password
  password: 密码

masquerade:
  type: proxy
  proxy:
    url: https://news.ycombinator.com/ ————要是密码没有输入正确，就会跳转到这个网站以进行迷惑
    rewriteHost: true
————————————————————————————————————————————————————————————————————————————
```



## 申请证书和密钥

如果你已经有证书和密钥了，可以直接跳到“复制证书到指定路径并设置权限”那一步



*   **用 `Certbot `申请证书，需要安装` Certbot `和` Nginx `插件**

```
dnf install -y epel-release
dnf install -y certbot python3-certbot-nginx
```



*   **申请证书**

```
certbot --nginx -d 域名
```

>   如果因为域名问题报错，可以去自己购买的域名网站查看域名是否相同



*   **查看申请证书位置**

里面应该有 `fullchain.pem` 和 `privkey.pem`。这就是你真正的证书路径。





## 复制证书到指定路径并设置权限



*   **把证书复制到` Hysteria 2 `的指定路径**

```
[root@localhost ~]# cp -L 证书路径/*.pem /etc/hysteria/
```



*   **修改证书权限，让`hysteria`自己也能访问**

```
[root@localhost ~]# chown hysteria:hysteria /etc/hysteria/config.yaml
[root@localhost ~]# chown hysteria:hysteria /etc/hysteria/证书 /etc/hysteria/私钥
[root@localhost ~]# chmod 600 /etc/hysteria/密钥
```



*   **查看是否运行**

```
[root@localhost ~]# systemctl start hysteria-server.service
[root@localhost ~]# systemctl status hysteria-server.service
```





## 添加防火墙策略



*   **添加防火墙策略，放行`udp443`接口**

```
[root@localhost ~]# firewall-cmd --permanent --zone=public --add-port=443/udp
[root@localhost ~]# firewall-cmd --reload
```



## 在 v2rayN 中添加节点



*   `v2rayN` → 配置项 → 添加 [`Hysteria 2`]
*   地址 = 服务器` IP`，端口 = `443`，密码 = 自己设置，`SNI `= 域名

添加后设置为活动，使用规则模式



## 验证



*   浏览器打开 https://ipinfo.io，显示的应该是服务器所在地的 IP



如果出现webRTC泄露原IP地址可以安装插件来防止

`adguard` 和`WebRTC Control`这两款插件都很好用



## 查看服务器日志

```
[root@localhost ~]# journalctl -u hysteria-server -n 20 --no-pager
```

验证是否成功连通





## 设置证书续期钩子



`/etc/hysteria/ `里的证书是副本，`certbot `续期并不代表也会更新它。

所以我们需要写一个脚本，让续期成功后自动复制并且重启` hysteria`。

    [root@localhost ~]# vim /etc/letsencrypt/renewal-hooks/deploy/hysteria.sh
    ————————————————————————————————————————————————————————————————————————————————
    #!/bin/bash
    cp -L 证书路径 /etc/hysteria/证书
    cp -L 证书路径 /etc/hysteria/密钥
    chown hysteria:hysteria /etc/hysteria/证书 /etc/hysteria/密钥
    chmod 600 /etc/hysteria/密钥
    systemctl restart hysteria-server
    ————————————————————————————————————————————————————————————————————————————————



*   **给下执行权限**

```
[root@localhost ~]# chmod +x /etc/letsencrypt/renewal-hooks/deploy/hysteria.sh
```





好，到目前为止，全部完工！☝️🤓

