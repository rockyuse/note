---
title: 'DHCP Server'
date: 2018-06-23 14:53:00
tags:
- NCTU
- DHCP
- NA
---

# DHCP Server

環境：Ubuntu 16.04 (DigitalOcean)

DHCP 的運作原理，可以參考[鳥哥](http://linux.vbird.org/linux_server/0340dhcp.php)

這邊簡單紀錄一下 Life Cycle 的部份

* Allocation：若一個 Client 尚未有任何租約，就向 Server 發送 allocation 的請求。
* Reallocation：若一個 Client 已經向 Server 申請租約，當 Client 重啟後，會向 DHCP Server 發送 reallocation，Server 會重新發送租約給 Client。整個過程類似 allocation，但速度更快。
* Normal Operation：當租約工作正常，且 Client 用分配的 IP 在 "main part" 做操作。
* Renewal：當經過租約的一定時間後，Client 會嘗試和 Server 要求 renew 租約，來延長這個 IP 的使用時間。
* Rebinding：若 Renewal 失敗(例如網路問題)，Client 會嘗試丟出 rebinding 的信息給任何可使用的 DHCP Server，來延長其租約。
* Release：Client 決定不要再用這個 IP 了，所以發送 release 給 Server，叫 Server 中止租約。

Life Cycle 的流程圖：([圖片來源](http://www.tcpipguide.com/free/t_DHCPLeaseLifeCycleOverviewAllocationReallocationRe.htm))

![](dhcplife.png)

## 安裝 DHCP Server

```shell
$ sudo apt-get install isc-dhcp-server
```

## 設定網卡介面

在 /etc/default/isc-dhcp-server 內，設定 INTERFACES

```
INTERFACES="lxcbr0"
```

## 設定 DHCP config

/etc/dhcp/dhcpd.conf ([Linux man page](https://linux.die.net/man/5/dhcpd.conf))

```
option domain-name "kaiiiz.nctucs.net";
option domain-name-server 8.8.8.8; // Client 預設的 DNS

default-lease-time 600;
max-lease-time 7200;

subnet 10.0.3.0 netmask 255.255.255.0 {
    range dynamic-bootp 10.0.3.2 10.0.3.100;
    option broadcast-address 10.0.3.255;
    option routers 10.0.3.1; // Client 預設的 router
}
```

設定完成後重啟 dhcp

```shell
$ sudo systemctl restart isc-dhcp-server
```

## 測試

```shell
$ lxc-attach -n hosta
```

進入 hosta

```shell
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 00:16:3e:7a:00:a0
          inet addr:10.0.3.107  Bcast:10.0.3.255  Mask:255.255.255.0
...
```

目前尚未和 Server 申請租約

```shell
$ dhclient
$ systemctl restart networking
```

申請租約

```shell
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 00:16:3e:7a:00:a0
          inet addr:10.0.3.3  Bcast:10.0.3.255  Mask:255.255.255.0
...
```

申請成功
