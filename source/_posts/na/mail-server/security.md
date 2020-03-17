---
title: 'Security'
tags:
- NCTU
- Mail
- NA
---

# Security

SASL // SMTP over TLS // SMTPs // POP3s // IMAPs

環境：Ubuntu 16.04 (DigitalOcean)

[為什麼需要它？](https://wiki.centos.org/zh-tw/HowTos/postfix_sasl#head-8efee0f83ff18b81caf3308ed554a94b7d6e5d00)

## SASL

SASL(Simple Authentication and Security Layer)：簡單來說，使用者發信前要先以帳號密碼認證，通過後才能寄信，這邊我們使用 dovecot-sasl 來設置

/etc/postfix/main.cf

```
...
# SASL
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination
```

啟動 SASL 及設定 SASL 支援非標準 E-mail Client 的認證動作

/etc/postfix/master.cf

```
smtp      inet  n       -       y       -       -       smtpd
  -o smtpd_sasl_auth_enable=yes
  -o broken_sasl_auth_clients=yes
```

創建 Unix socket 所在的文件夾

/etc/dovecot/conf.d/10-master.conf

```
service auth {
  # auth_socket_path points to this userdb socket by default. It's typically
  # used by dovecot-lda, doveadm, possibly imap process, etc. Its default
  # permissions make it readable only by root, but you may need to relax these
  # permissions. Users that have access to this socket are able to get a list
  # of all usernames and get results of everyone's userdb lookups.
#  unix_listener auth-userdb {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }
...
```

指定授權機制，即在 SMTP Command 中 auth 後面顯示的內容

/etc/dovecot/conf.d/10-auth.conf

```
auth_mechanisms = plain login
```

重啟服務

```shell
$ systemctl restart dovecot
$ systemctl restart postfix
```

### 測試

```shell
$ telnet localhost 25
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
220 kaiiiz.nctucs.net ESMTP Postfix (Ubuntu)
```

有出現下面兩行表示設置成功

```
ehlo localhost
...
250-AUTH PLAIN LOGIN
250-AUTH=PLAIN LOGIN
...
```

> ehlo 是 helo 的擴展，即extend helo，可以支援用戶認證。

## SMTP over TLS

### 生成憑證

這邊用 letsencrypt 生 key，比較方便

```shell
$ sudo letsencrypt certonly --standalone -d kaiiiz.nctucs.net
```

key 會存在 /etc/letsencrypt/live/kaiiiz.nctucs.net/ 這個目錄中

若遇到下面這個問題

```
Problem binding to port 80: Could not bind to IPv4 or IPv6.
```

記得先停用佔著 80 port 的程式（e.g. Apache）

### 設定 Postfix

把註解改掉，然後改一下 key 的位置，再改一下 TLS 的設定

/etc/postfix/main.cf

```
...
smtpd_tls_cert_file=/etc/letsencrypt/live/kaiiiz.nctucs.net/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/kaiiiz.nctucs.net/privkey.pem
...
smtp_use_tls=yes
smtpd_use_tls=yes
smtp_tls_note_starttls_offer=yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
...
```

### 測試

```shell
$ telnet localhost 25
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
220 kaiiiz.nctucs.net ESMTP Postfix (Ubuntu)
```

應該會看到下面這個

```
ehlo localhost
...
250-STARTTLS
...
```

## SMTPs

/etc/postfix/master.cf

應該會看到下面這行

```
...
#smtps     inet  n       -       y       -       -       smtpd
...
```

把下面的內容設置一下

```
#smtps     inet  n       -       y       -       -       smtpd
465     inet  n       -       y       -       -       smtpd
  -o smtpd_tls_security_level=encrypt
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

### 測試

看有沒有 465

```shell
$ sudo netstat -tlnp | grep 465
tcp        0      0 0.0.0.0:465             0.0.0.0:*               LISTEN      19843/master
tcp6       0      0 :::465                  :::*                    LISTEN      19843/master
```

看能不能透過 465 連進去

```shell
$ openssl s_client -connect localhost:465
```

## 關閉 SMTP 登入

既然已經有較安全的 SMTP with TLS 及 SMTPs，我們可以把 SMTP 純文字的 SASL 關掉，保證連線的安全。

在 /etc/postfix/master.cf 下，加入設定：

```
smtp      inet  n       -       y       -       -       smtpd
  -o smtpd_tls_auth_only=yes
```

> smtpd_tls_auth_only=yes：強迫 SASL 驗證採用 TLS 並禁止純文字驗證發生

### 測試

```shell
$ telnet localhost 25
...
ehlo localhost
```

如果一切正常，我們應該會看見伺服器提供 STARTTLS，而且因為我們指定 'smtpd_tls_auth_only = yes'，純文字 SASL 驗證（AUTH PLAIN LOGIN 及 AUTH=PLAIN LOGIN）已不再被支援。

## POP3s / IMAPs

填一下剛剛用 letsencrypt 生成 key 的位置

/etc/dovecot/conf.d/10-ssl.conf

```
...
# SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
ssl = required

# PEM encoded X.509 SSL/TLS certificate and private key. They're opened before
# dropping root privileges, so keep the key file unreadable by anyone but
# root. Included doc/mkcert.sh can be used to easily generate self-signed
# certificate, just make sure to update the domains in dovecot-openssl.cnf
ssl_cert = </etc/letsencrypt/live/kaiiiz.nctucs.net/fullchain.pem
ssl_key = </etc/letsencrypt/live/kaiiiz.nctucs.net/privkey.pem
...
```

重啟 dovecot

```shell
$ sudo systemctl restart dovecot
```

### 強制使用者開啟 SSL

/etc/dovecot/conf.d/10-auth.conf

```
disable_plaintext_auth = yes
```

> disable_plaintext_auth：停用一切純文字的驗證方法

### 測試

```shell
$ sudo netstat -tlnp
```

看看 port 995 (POP3s) 和 993 (IMAPs) 有沒有 work

## 後記

備份一些檔案，可以當作參考

* [postfix](https://gist.github.com/3f4505f446c5c24b7db4fe26f39f38f8.git)
* [dovecot](https://gist.github.com/536e0a6502aa7c9a5ed51fdef382bd46.git)