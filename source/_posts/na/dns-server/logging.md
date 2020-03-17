---
title: 'Logging - DNS Server'
tags:
- NCTU
- NA
- DNS
---

# Logging

為了讓資料更有結構，我們把 log 的設定放在 /etc/bind/**named.conf.logging**，並在 **named.conf** 中 include 它：

```
include "/etc/bind/named.conf.logging";
```

## Config

/etc/bind/named.cong.logging：

```
logging
{
    channel default-log {
        file "/var/log/named/named.log" versions unlimited size 1m;
        severity info;
        print-time yes;
    };
    channel lamer-log {
        file"/var/log/named/named.log" versions unlimited size 1m;
        severity info;
        print-severity yes;
        print-time yes;
        print-category yes;
    };
    channel query-log {
        file "/var/log/named/named.log" versions unlimited size 1m;
        severity info;
        print-time yes;
    };
    channel security-log {
        file"/var/log/named/named.log" versions unlimited size 1m;
        severity info;
        print-severity yes;
        print-time yes;
        print-category yes;
    };
    category lame-servers { lamer-log; };
    category security{ security-log;};
    category queries { query-log;};
    category default { default-log;};
};
```

我們把 log 都放在 **/var/log/named/named.log** 下，當 size 達到 1m 時就執行 rotate，rotate 的數量沒有限制。

## 測試

到 /var/log/named 下看是否成功 rotate

```
/var/log/named $ l
total 4.1M
drwxrwxr-x 2 bind bind   4.0K May  3 07:37 .
drwxrwxr-x 9 root syslog 4.0K May  3 06:25 ..
-rw-r--r-- 1 bind bind    36K May  3 17:44 named.log
-rw-r--r-- 1 bind bind   1.1M May  3 13:04 named.log.0
-rw-r--r-- 1 bind bind   1.1M Apr 24 18:23 named.log.1
-rw-r--r-- 1 bind bind   1.1M Apr 24 18:20 named.log.2
-rw-r--r-- 1 bind bind   1.1M Apr 25 01:42 named.log.3
```

> 我們用來生log的方法：寫一個 SHELL script 瘋狂 query server 看會不會成功 rotate XD

## 問題與解法

### log 權限問題

因為作業要求存log的位置有權限的問題，要直接改 server 系統的資料又有點噁心

況且 bind 已經告訴我們有哪幾個資料夾可以直接使用了，於是我們就改變了log的位置到 bind 幫我們設定好的位址：**/var/log/named**

後來 demo 也沒有要求一定要存在哪

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
    │   named.conf.logging
    │   named.conf.options
    │   named.conf.default-zones
```