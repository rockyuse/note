---
title: 'Webmail、Alias'
tags:
- NCTU
- Mail
- NA
---

# Webmail、Alias

RainLoop // virtual alias

環境：Ubuntu 16.04 (DigitalOcean)

## Webmail

詳細的教學可以參考[這篇](https://www.linuxbabe.com/mail-server/install-rainloop-webmail-ubuntu-16-04)

### 安裝

我用的是 Apache

```shell
$ sudo apt install apache2 php7.0 libapache2-mod-php7.0 php7.0-curl php7.0-xml
```

```shell
$ mkdir rainloop
$ cd rainloop
$ curl -s http://repository.rainloop.net/installer.php | php
$ cd ..
$ sudo mv rainloop /var/www/
$ sudo chown www-data:www-data /var/www/rainloop/ -R
```

### Apache

```shell
$ sudo vim /etc/apache2/sites-available/rainloop.conf
```

```
<VirtualHost *:80>
  ServerName kaiiiz.nctucs.net
  DocumentRoot "/var/www/rainloop/"
  ServerAdmin andy@kaiiiz.nctucs.net

  ErrorLog "/var/log/apache2/rainloop_error_log"
  TransferLog "/var/log/apache2/rainloop_access_log"

  <Directory />
    Options +Indexes +FollowSymLinks +ExecCGI
    AllowOverride All
    Order deny,allow
    Allow from all
    Require all granted
  </Directory>

RewriteEngine on
RewriteCond %{SERVER_NAME} =kaiiiz.nctucs.net
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

```shell
$ sudo a2ensite rainloop.conf
```

```shell
$ sudo systemctl reload apache2
```

### HTTPS

```shell
$ sudo apt install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt update
$ sudo apt install certbot python-certbot-apache
```

```shell
$ sudo certbot --apache --agree-tos --email andy@kaiiiz.nctucs.net -d kaiiiz.nctucs.net
```

### 調整

設定完後到 kaiiiz.nctucs.net/?admin 改一下密碼，之後可以調整一下安全性什麼的。

## Forward

詳細可參考[鳥哥](http://linux.vbird.org/linux_server/0380mail.php#postfix_alias)

### virtual alias

可以用正則表達式處理別名

/etc/postfix/main.cf

```
# virtual_alias
virtual_alias_maps = regexp:/etc/postfix/virtual
```

/etc/postfix/virtual

```
/^.*demo.*@kaiiiz.nctucs.net$/ hw4user
/^(.*)+.*@kaiiiz.nctucs.net$/ $1@kaiiiz.nctucs.net
```

* .\*demo.\*@{YourDomain} alias to hw4user
* {USER}+.*@{YourDomain} alias to {USER}
  * e.g., hw4user+abc@{YourDomain} send to hw4user
