# Let's 免费 HTTPS 证书

主要使用 certbot 工具生成。

## 1. 下载 certbot

```sh
git clone https://github.com/certbot/certbot
cd certbot
./certbot-auto --help
```

## 2. 生成免费证书

执行以下命令即可生成证书：

```sh
# 语法如下 ./certbot-auto certonly --webroot --agree-tos -v -t --email 邮箱地址 -w 网站根目录 -d 网站域名
# 具体如下
./certbot-auto certonly --webroot --agree-tos -v -t --email xueduan.li@gmail.com -w /usr/local/projects/blog/html -d lixueduan.com


./certbot-auto certonly --webroot --agree-tos -v -t --email 1033256636@qq.com -w /usr/local/projects/html -d zzra.com
```

**注意** 这里 默认会自动生成验证文件到 `/**网站根目录**/.well-known/acme-challenge` 文件夹，然后 shell 脚本会对应的访问 `**网站域名**/.well-known/acme-challenge `是否存在来确定你对网站的所属权。

比如：我的域名是 **lixueduan.com** 那我就得保证域名下面的 **.well-known/acme-challenge/** 目录是可访问的

如果返回正常就确认了你对这个网站的所有权，就能顺利生成，完成后这个目录会被清空。

## 3. 测试

如果上面的步骤正常 shell 脚本会展示如下信息：

```sh
- Congratulations! Your certificate and chain have been saved at
/etc/letsencrypt/live/网站域名/fullchain.pem
...
```

## 4. 生成 dhparams

使用 openssl 工具生成 dhparams

```bash
openssl dhparam -out /etc/ssl/certs/dhparams.pem 2048
```

## 5. 配置 Nginx

打开 nginx server 配置文件加入如下设置：

```nginx
listen 443

ssl on;
ssl_certificate /etc/letsencrypt/live/网站域名/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/网站域名/privkey.pem;
ssl_dhparam /etc/ssl/certs/dhparams.pem;
ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers HIGH:!aNULL:!MD5;
```

然后重启 nginx 服务就可以了

## 6. 强制跳转 https

https 默认是监听 443 端口的，没开启 https 访问的话一般默认是 80 端口。如果你确定网站 80 端口上的站点都支持 https 的话加入下面的配件可以自动重定向到 https

```nginx
server {
    listen 80;
    server_name your.domain.com;
    return 301 https://$server_name$request_uri;
}
```

## 7. 证书更新

免费证书只有 90 天的有效期，到时需要手动更新 renew。刚好 Let’s encrypt 旗下还有一个 [Let’s monitor](https://letsmonitor.org/) 免费服务，注册账号添加需要监控的域名，系统会在证书马上到期时发出提醒邮件，非常方便。收到邮件后去后台执行 renew 即可，如果提示成功就表示 renew 成功

```bash
./certbot-auto renew
```







## 新版

certbot-auto申请证书时发现如下提示;certbot-auto被弃用了

```sh
Skipping bootstrap because certbot-auto is deprecated on this system.
Your system is not supported by certbot-auto anymore.
Certbot cannot be installed.
Please visit https://certbot.eff.org/ to check for other alternatives.
```

certbot-auto不再支持所有的操作系统！根据作者的说法，certbot团队认为维护certbot-auto在几乎所有流行的UNIX系统以及各种环境上的正常运行是一项繁重的工作，加之certbot-auto是基于python 2编写的，而python 2即将寿终正寝，将certbot-auto迁移至python 3需要大量工作，这非常困难，因此团队决定放弃certbot-auto的维护。

既然如此，现在我们还能继续使用certbot吗？certbot团队使用了基于snap的新的分发方法。

```sh
https://github.com/certbot/certbot/issues/8535
```



详细信息见[官方文档](https://certbot.eff.org/docs/using.html)

新版手动生成流程如下：

```sh
$ certbot certonly --manual
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): example.com # 1.这里输入域名
Requesting a certificate for example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
# 2. 这里在服务器根目录对应位置加上对应文件 用于校验域名所属
Create a file containing just this data:

ISGb__SljshkeZwofJEAGdp7fhJuedWNhWf0wvOXslc.VgoFElVNd4xMqN9HaHffeTp4xRept_XUAcM8j2gllX0

And make it available on your web server at this URL:

http://example.com/.well-known/acme-challenge/ISGb__SljshkeZwofJEAGdp7fhJuedWNhWf0wvOXslc

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
# 3.文件添加后回车即可
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2021-09-28.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```



```sh
certbot certonly -d *.zzra.com --manual --preferred-challenges dns
```



* -d 指定域名
* --manual 手动模式
*  --preferred-challenges 指定校验类型 dns、http
* dns 则是添加dns记录
  * http为在站点根目录防止文件
  * 其中泛域名必须使用 dns 方式校验

* email 证书到期时的提醒邮箱




