---
title: '反垃圾郵件'
tags:
- NCTU
- Mail
- NA
---

# 反垃圾郵件

RBL // Greylisting

環境：Ubuntu 16.04 (DigitalOcean)

## RBL

RBL(Real-time blacklists)

/etc/postfix/main.cf

```
# RBL
smtpd_sender_restrictions = reject_rbl_client zen.spamhaus.org
```

重啟服務

```shell
$ sudo systemctl restart postfix
```

## Greylisting

原理：第一次收到郵件就先退回這封郵件，等一段時間後如果又收到同樣的郵件就接收這封郵件。（正常的 Mail Server 會嘗試 Retry）

### 安裝

```shell
$ sudo apt-get install postgrey
```

### 設定

檢查一下 postgrey 聽哪個 port

```shell
$ sudo netstat -tlnp | grep postgrey
```

/etc/postfix/main.cf

```
smtpd_recipient_restrictions =
    ...
    check_policy_service inet:127.0.0.1:10023
```

重啟服務

```shell
$ sudo systemctl restart postfix
```

### 拒收時限

預設預設拒收郵件的時間是5分鐘，可以透過 --delay 調整

/etc/default/postgrey

```
POSTGREY_OPTS="--inet=127.0.0.1:10023 --delay=60"
```