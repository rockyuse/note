---
title: 'View - DNS Server'
tags:
- NCTU
- NA
- DNS
---

# View

透過 View 來控制回應的 Record。

## ACL

我們需要區別 IP 是來自 bsd 工作站或 linux 工作站的 query，所以我們先把 IP 歸類好：

named.conf.local

```
acl "linux" {
    140.113.235.151;
    140.113.235.152;
    140.113.235.153;
    140.113.235.154;
};
acl "bsd" {
    140.113.235.131;
    140.113.235.132;
    140.113.235.133;
    140.113.235.134;
};
```

## Local Zone with View

接著分別區分在 linux 的 view：

named.conf.local

```
view "internal-linux" {
    match-clients { "linux"; };

    zone "kaiiiz.nctucs.net" IN {
        type master;
        file "zone/db.kaiiiz.nctucs.net";
    };
    zone "184.229.35.in-addr.arpa" {
        type master;
        file "zone/db.35.229.184";
    };

    zone "sub.muller.nctucs.net" IN {
        type master;
        file "zone/db.sub.muller.nctucs.net.internal-linux";
        notify yes;
    };
    zone "sub.kaiiiz.nctucs.net" IN {
        type slave;
        file "slaves/db.sub.kaiiiz.nctucs.net.linux";
        masters { 35.229.252.239; };
    };
};
```

以及在 bsd 的 view：

named.conf.local

```
view "internal-bsd" {
    match-clients { "bsd"; };

    zone "kaiiiz.nctucs.net" IN {
        type master;
        file "zone/db.kaiiiz.nctucs.net";
    };
    zone "184.229.35.in-addr.arpa" {
        type master;
        file "zone/db.35.229.184";
    };

    zone "sub.muller.nctucs.net" IN {
        type master;
        file "zone/db.sub.muller.nctucs.net.internal-bsd";
        notify yes;
    };
    zone "sub.kaiiiz.nctucs.net" IN {
        type slave;
        file "slaves/db.sub.kaiiiz.nctucs.net.bsd";
        masters { 35.229.252.239; };
    };
};
```

還有其他的 query：

named.conf.local

```
view "otherwise" {
    match-clients { any; };

    zone "kaiiiz.nctucs.net" IN {
        type master;
        file "zone/db.kaiiiz.nctucs.net";
    };
    zone "184.229.35.in-addr.arpa" {
        type master;
        file "zone/db.35.229.184";
    };

    zone "sub.kaiiiz.nctucs.net" IN {
        type slave;
        file "slaves/db.sub.kaiiiz.nctucs.net";
        masters { 35.229.252.239; };
    };
    zone "sub.muller.nctucs.net" IN {
        type master;
        file "zone/db.sub.muller.nctucs.net";
        notify yes;
    };
};
```

## Default Zone with View

別忘了 **named.conf.default-zones** 裡面的也要改：

```
view "any" {
    match-clients { any; };
    zone "." {
        type hint;
        file "/etc/bind/db.root";
    };

    // be authoritative for the localhost forward and reverse zones, and for
    // broadcast zones as per RFC 1912

    zone "localhost" {
        type master;
        file "/etc/bind/db.local";
    };

    zone "127.in-addr.arpa" {
        type master;
        file "/etc/bind/db.127";
    };

    zone "0.in-addr.arpa" {
        type master;
        file "/etc/bind/db.0";
    };

    zone "255.in-addr.arpa" {
        type master;
        file "/etc/bind/db.255";
    };
};
```

## Zone files

這裡我們多了幾個 zone file：

* zone/db.sub.muller.nctucs.net.**internal-linux**
* zone/db.sub.muller.nctucs.net.**internal-bsd**
* slaves/db.sub.kaiiiz.nctucs.net.bsd
* slaves/db.sub.kaiiiz.nctucs.net.linux

slaves 的兩個新的 zone files 因為是交由隊友託管所以不用管他，master 的兩個新的 zone files 基本上檔案的長相都跟 db.sub.muller.nctucs.net 一樣，只是可以設定來自不同 view 的 query 該回應哪個對應的 A record 如此

db.sub.muller.nctucs.net.internal-linux：

```
$TTL 600
@       IN      SOA     dns.kaiiiz.nctucs.net. admin.kaiiiz.nctucs.net. (
                        2018041607
                        1800
                        900
                        1W
                        1D )
@ IN NS dns.kaiiiz.nctucs.net.
@ IN NS dns.muller.nctucs.net.

demo IN CNAME ws.sub.muller.nctucs.net.

ns1     IN      A       192.168.1.1
sub     IN      A       192.168.1.2
test    IN      A       192.168.1.3
ws.sub.muller.nctucs.net. IN A 140.113.235.151
```

db.sub.muller.nctucs.net.internal-bsd：

```
$TTL 600
@       IN      SOA     dns.kaiiiz.nctucs.net. admin.kaiiiz.nctucs.net. (
                        2018041607
                        1800
                        900
                        1W
                        1D )
@ IN NS dns.kaiiiz.nctucs.net.
@ IN NS dns.muller.nctucs.net.

demo IN CNAME ws.sub.muller.nctucs.net.

ns1     IN      A       192.168.1.1
sub     IN      A       192.168.1.2
test    IN      A       192.168.1.3
ws.sub.muller.nctucs.net. IN A 140.113.235.131
```

可以觀察到與 db.sub.muller.nctucs.net.internal-bsd 的區別只在於 dig ws.sub.muller.nctucs.net **回應的 A 會不同**

## 測試

### View 測試

在系計中的 linux 和 bsd 工作站和外面 dig ws.sub.muller.nctucs.net 應該要回應不同的 A record

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
│   │   │  db.sub.muller.nctucs.net.internal-linux
│   │   │  db.sub.muller.nctucs.net.internal-bsd
│   │
│   └───slaves
│       │  db.sub.kaiiiz.nctucs.net
│       │  db.sub.kaiiiz.nctucs.net.linux
│       │  db.sub.kaiiiz.nctucs.net.bsd
│   
└───etc/bind/
    │   named.conf
    │   named.conf.local
    │   named.conf.logging
    │   named.conf.options
    │   named.conf.default-zones
```