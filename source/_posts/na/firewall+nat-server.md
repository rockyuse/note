---
title: 'Firewall & NAT Server'
date: 2018-06-23 16:11:00
tags:
- NCTU
- iptables
- NA
---

# Firewall & NAT Server

環境：Ubuntu 16.04 (DigitalOcean)

防火牆相關知識，可以參考[鳥哥](http://linux.vbird.org/linux_server/0250simple_firewall.php)

## 清理 table

```shell
iptables -F
iptables -t nat -F
iptables -X
iptables -Z
```

> -F: Flush chains
> 
> -t: Table
> 
> -X: Delete chains
> 
> -Z: Zero the packet and byte counters in all chains

## 設定預設 policy

```shell
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
```

> -P: Policy

## 基本設置

允許 localhost 的封包進入本機

```shell
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -i $EXTIF -m state --state RELATED,ESTABLISHED -j ACCEPT
```

> -A: Append one or more rules to the end of the chain
> 
> -i: Interface name
> 
> -m: Load match module
> 
> -j: Target of the rule(e.g. what to do if the packet matches it)
> 
> ESTABLISHED: 有效的封包，且此封包屬於一個已建立的連結，這個連結的兩端都已經有數據發送。
> 
> RELATED: 封包正在建立一個新的連結，此連結是與一個已建立的連結相關的。例如，FTP data transfer。

## 拒絕 Client

### Reject

拒絕並回應 TCP RST / ICMP host unreachable

```shell
iptables -A INPUT -s $ip -p tcp -m tcp --dport 222 -j REJECT --reject-with tcp-reset
```

> -p: Protocal
> 
> --dport: 是 tcp module 的一個功能，用來限制目標的埠口號碼。
> 
> --reject-with: 以什麼方法reject，(e.g. tcp-reset, icmp-host-unreachable, icmp-port-unreachable)

### Drop

拒絕且不回應任何訊息

拒絕目標為 222 port 的封包

```shell
iptables -A INPUT -p tcp -m tcp --dport 222 -j DROP
```

拒絕 ICMP ECHO-REPLY

```shell
iptables -A OUTPUT -p icmp -m icmp --icmp-type 0 -j DROP
```

## NAT Routing

相關知識可以參考：[鳥哥](http://linux.vbird.org/linux_server/0250simple_firewall.php#nat)

1. 先經過 NAT table 的 PREROUTING 鏈；
2. 經由路由判斷確定這個封包是要進入本機與否，若不進入本機，則下一步；
3. 再經過 Filter table 的 FORWARD 鏈；
4. 通過 NAT table 的 POSTROUTING 鏈，最後傳送出去。

簡單理解：POSTROUTING 在修改**來源IP** (SNAT)，PREROUTING 則在修改**目標IP** (DNAT)。

### 讓 Server 有 FORWARD 的功能

/etc/sysctl.conf，註解掉

```
net.ipv4.ip_forward=1
```

### POSTROUTING

```shell
iptables -t nat -A PREROUTING -s $ip -i $EXTIF -p tcp -m tcp --dport 222 -j DNAT --to-destination 10.0.3.150:22
```

> -s: Source Address(Network name, hostname, network IP address(with /mask), plain IP address)

### PREROUTING

```shell
iptables -t nat -A POSTROUTING -s $INTIF -o $EXTIF -j MASQUERADE
```

> -o: Out interface

### MASQUERADE V.S. SNAT

MASQUERADE 的作用是，從服務器的網卡上，自動獲取當前ip地址來做NAT
SNAT 則是直接修改一個或多個來源IP

### FOWARD

* FORWARD 政策允許系統管理員控制封包在區網內部的傳送。
* 讓防火牆之後的系統存取內部網路。
* Router 會將某個區域網路節點上的封包，透過 $INTIF 裝置，送到目的地節點去。

詳細可參考：[FORWARD 與 NAT 規則](http://web.mit.edu/rhel-doc/4/RH-DOCS/rhel-sg-zh_tw-4/s1-firewall-ipt-fwd.html)

```
iptables -A FORWARD -o $INTIF -j ACCEPT
iptables -A FORWARD -i $INTIF -j ACCEPT
```

i.e. 若把 -o 那行後方改為 DROP，則內網要出去的封包即會被丟棄。

## 指令

### 列出 iptables

```shell
sudo iptables -L -nv
```

> -L: List all rules
> 
> -n: Numeric output(IP addresses and port numbers will be printed in numeric format)
> 
> -v: Verbose output(Show interface name, rule options, TOS(Type of Service) mask)

```shell
sudo iptables -t nat -L -nv
```

> -t: Table

### 備份 rules

```shell
sudo iptables-save
```

列出目前設定

```shell
sudo iptables-save > filename
```

可以這樣存成一個檔案

### 還原 rules

```shell
sudo iptables-restore < filename
```

## 後記

有興趣可以參考完整的 sh 檔：[NA homework 3 - firewall](https://gist.github.com/1f69fbac0bb557de360be2fac294f6e3.git)