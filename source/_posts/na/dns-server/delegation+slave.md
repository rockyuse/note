---
title: 'Delegation & Slave - DNS Server'
tags:
- NCTU
- NA
- DNS
---

# Delegation & Slave

用 Sub-domain 來做託管與備援，讓兩個 DNS Server 間同步 zone file。

## Delegation

在 named.conf.local 新增一個 zone

```
zone "sub.muller.nctucs.net" IN {
    type master;
    file "zone/db.sub.muller.nctucs.net";
    notify yes;
};
```

這是隊友託管給我們的 domain，所以我們就是**這個 domain 的 master**

> **notify** 的意思是每次重啟一次 bind 的服務，就通知隊友的 slave 該來同步資料了！

因為我們是 master ，所以也要新增這個 zone 的 zone file：

/var/cache/bind/zone/db.sub.muller.nctucs.net

```
$TTL 600
@       IN      SOA     dns.kaiiiz.nctucs.net. admin.kaiiiz.nctucs.net. (
                        2018042407
                        1800
                        900
                        1W
                        1D )
@ IN NS dns.kaiiiz.nctucs.net.
@ IN NS dns.muller.nctucs.net.

ns1     IN      A       192.168.1.1
sub     IN      A       192.168.1.2
test    IN      A       192.168.1.3
```

因為這個 domain 是給兩個 server 管理的，所以也要加上兩個 NS record

至於後面可以自己新增一些 A record，方便之後的測試。

## Slave

在 named.conf.local 新增一個 zone

```
zone "sub.kaiiiz.nctucs.net" IN {
    type slave;
    file "slaves/db.sub.kaiiiz.nctucs.net";
    masters { 35.229.252.239; };
};
```

這是我們要給隊友託管的 domain，並且讓我們的 server(dns.kaiiiz.nctucs.net) 做為隊友的備援

file 那行是設定從隊友的 server 同步過來的資料的儲存位址，也就是 /var/cache/bind/slaves/db.sub.kaiiiz.nctucs.net

因為這個檔案就是從隊友那裡同步過來的，所以 **不需要** 建立 db.sub.kaiiiz.nctucs.net 這個 zone file

## 測試

### Delegation

```shell
$ dig test.sub.muller.nctucs.net

...

;; QUESTION SECTION:
;test.sub.muller.nctucs.net.	IN	A

;; ANSWER SECTION:
test.sub.muller.nctucs.net. 581	IN	A	192.168.1.3

;; AUTHORITY SECTION:
sub.muller.nctucs.net.	581	IN	NS	dns.muller.nctucs.net.
sub.muller.nctucs.net.	581	IN	NS	dns.kaiiiz.nctucs.net.

;; ADDITIONAL SECTION:
dns.muller.nctucs.net.	598205	IN	A	35.229.252.239

...
```

有拿到 A record 代表沒問題

### Slave

```shell
$ dig test.sub.kaiiiz.nctucs.net @dns.kaiiiz.nctucs.net

...

;; QUESTION SECTION:
;test.sub.kaiiiz.nctucs.net.	IN	A

;; ANSWER SECTION:
test.sub.kaiiiz.nctucs.net. 600	IN	A	35.229.252.239

;; AUTHORITY SECTION:
sub.kaiiiz.nctucs.net.	600	IN	NS	dns.muller.nctucs.net.
sub.kaiiiz.nctucs.net.	600	IN	NS	dns.kaiiiz.nctucs.net.

;; ADDITIONAL SECTION:
dns.kaiiiz.nctucs.net.	600	IN	A	35.229.184.206

;; Query time: 6 msec
;; SERVER: 35.229.184.206#53(35.229.184.206)

...
```

後面的 server 來自 dns.kaiiiz.nctucs.net ，代表 slave 有設定成功

## 小結

目前我們重要檔案的架構如下：

```
/
└───var/cache/bind/
│   │
│   └───zone
│   │   │  db.kaiiiz.nctucs.net
│   │   │  db.35.229.184
│   │   │  db.sub.muller.nctucs.net
│   │
│   └───slaves
│       │  db.sub.kaiiiz.nctucs.net
│
└───etc/bind/
    │   named.conf
    │   named.conf.local
    │   named.conf.options
    │   named.conf.default-zones
```