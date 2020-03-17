---
title: '反詐騙郵件'
tags:
- NCTU
- Mail
- NA
---

# 反詐騙郵件

SPF // SRS // DKIM // DMARC

環境：Ubuntu 16.04 (DigitalOcean)

## SPF

SPF(Sender Policy Framework)，讓某 domain 授權特定 IP Address 代表其發信，MTA收到後會驗證該地址 domain 的 SPF 紀錄的 range 是否跟信上的 IP Address 相同，目的是在確認電子郵件確實是由網域授權的郵件伺服器寄出

/var/cache/bind/zone/db.kaiiiz.nctucs.net

```
; SPF record
kaiiiz.nctucs.net. IN TXT "v=spf1 +a +mx -all"
kaiiiz.nctucs.net. IN SPF "v=spf1 +a +mx -all"
```

> v=spf1：spf 所使用的版本
> 
> +：Pass
> 
> -：Fail，e.g. -all 表示除了有條列出來的主機允許其他都拒絕，標式為 Hard Fail 不會接受該信件
> 
> ~：Softfail，e.g. ~all 表示除了有條列出來的主機允許其他都拒絕，標式為 Soft Fail 還是接收了該信件
> 
> ?：Neutral
> 
> a, mx：表示比對 DNS 記錄中的 A 及 MX record

DNS 的架設可以參考[這篇](https://kaiiiz.nctu.me/2018/05/03/dns-server-1/)

### SPF check

檢查進來的信有沒有符合 SPF

官方的[教學](https://help.ubuntu.com/community/Postfix/SPF)

```shell
$ sudo apt-get install postfix-policyd-spf-python
```

/etc/postfix/main.cf

```
# SPF
policy-spf_time_limit = 3600s
...
smtpd_recipient_restrictions =
     ...
     check_policy_service unix:private/policy-spf
     ...
```

/etc/postfix/master.cf

```
policy-spf  unix  -       n       n       -       -       spawn
     user=nobody argv=/usr/bin/policyd-spf
```

重啟服務

```shell
$ sudo systemctl restart postfix
```

## SRS

SPF 是嘗試防範電子郵件偽造的機制，方法是查找與電子郵件 _FROM: (訊息封寄件者)_ 位址網域相關聯的特殊 _TXT_ 記錄。

想像一下，當你的機器 forward 別人的 mail 時，E-mail 上的 _FROM:_ 應為原信件的 domain name

![](srsdetail_1.png)

但 SPF check 查找發信的人的 DNS，會發現轉寄信的人不符合 DNS 上設定的 TXT record，因此沒有通過 SPF check

![](srsdetail_3.png)

寄件者重寫機制 (SRS) 提供此問題的解決方案。SRS 使用寄件者的網域，將原始寄件者的位址封裝至新位址中。只有轉寄者自己的網域會顯露供 SPF 檢查

詳細可參考：[這裡](http://www.open-spf.org/srs)以及[這裡](https://docs.oracle.com/cd/E19957-01/820-0514/geyun/index.html)

### 安裝

```shell
$ sudo apt install postsrsd
```

/etc/postfix/main.cf

```
# PostSRSd settings.
sender_canonical_maps = tcp:localhost:10001
sender_canonical_classes = envelope_sender
recipient_canonical_maps = tcp:localhost:10002
recipient_canonical_classes= envelope_recipient,header_recipient
```

重啟服務

```shell
$ sudo systemctl restart postfix
```

## DKIM

DKIM 的原理是用公私鑰來驗證寄件者的真偽。Public Key 放在 DNS TXT Record，而 Private Key 存放在 Server，而每封信都帶有 DKIM-Signature。這樣只要驗證 DKIM-Signature 是否和 Public key 相符就可以知道郵件的真偽。

```shell
$ sudo apt-get install opendkim opendkim-tools
```

/etc/default/opendkim

```
SOCKET="inet:8891@localhost"
```

生成 Public key 及 Private key

```shell
$ opendkim-genkey -t -s mail -d kaiiiz.nctucs.net
```

會生成出 mail.txt 及 mail.private，把 mail.txt 的內容放到 DNS 中，把 mail.private 複製到 postfix 的資料夾方便管理

```shell
$ sudo cp mail.private /etc/postfix/dkim.key
```

/etc/opendkim.conf

```
Domain                  kaiiiz.nctucs.net
KeyFile                 /etc/postfix/dkim.key
Selector                mail
```

### DKIM check

檢查進來的信有沒有符合 DKIM

/etc/postfix/main.cf

```
# DKIM
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

重啟服務

```shell
$ sudo systemctl restart postfix
$ sudo systemctl restart opendkim
```

## DMARC

[為什麼要DMARC？](https://support.google.com/a/answer/2466580?hl=zh-Hant&ref_topic=2759254)

使用方法很簡單，加一個 txt record 就好了

/var/cache/bind/zone/db.kaiiiz.nctucs.net

```
; DMARC
_dmarc IN TXT "v=DMARC1; p=reject; rua=mailto:postmaster@mail.kaiiiz.nctucs.net"
```

### DMARC check

檢查進來的信有沒有符合 DMARC

/etc/postfix/main.cf

```
smtpd_milters = ...,inet:localhost:83682
non_smtpd_milters = ...,inet:localhost:83682
```

重啟服務

```shell
$ sudo systemctl restart postfix
```

## 測試

寄一封信給 google，查看郵件原始檔，看是否有符合 SPF、DKIM、DMARC 的驗證

```shell
$ echo "This is the body" | mutt -s "Test mail" name@mail_address
```

從 google 寄信進來，看 SPF、DKIM、DMARC 有沒有 check