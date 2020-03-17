---
title: 'LXC'
tags:
- lxc
---

# LXC

https://linuxcontainers.org/

## 前言

環境：Ubuntu 16.04 (DigitalOcean)

## 安裝

```shell
$ sudo apt-get install lxc
```

安裝完後可以用 ifconfig 看一下有沒有多出一個 lxcbr0 這個網卡，若沒有有可能是不同服務跟 lxc 這個衝突了，我遇到的是 DNS Server 裝的 bind9 跟 lxc 衝突

解決方法就是先 stop bind9 和 lxc 再 start lxc 再 start bind9

## Unprivilged User

詳細教學：[LXC- Getting started](https://linuxcontainers.org/lxc/getting-started/)

### /etc/lxc/lxc-usernet

```
andy veth lxcbr0 2
```

### subuid & subgid

```shell
$ grep 'andy' /etc/sub?id
/etc/subgid:andy:165536:65536
/etc/subuid:andy:165536:65536
```

### config

```shell
$ mkdir -p ~/.config/lxc
$ cp /etc/lxc/default.conf ~/.config/lxc
```

在 ~/.config/lxc 內加入

```
lxc.id_map = u 0 165536 65536
lxc.id_map = g 0 165536 65536
```

後面兩個改成剛剛查詢到的 uid 和 gid 的值

## Create Container

可以用

```shell
$ lxc remote list
```

列出 image 的儲藏庫，找到需要的再用

```shell
$ lxc-create -t download -n hosta -- -d ubuntu -r xenial -a amd64
```

建立 Container

## Open Container

```shell
$ lxc-start -n hosta -d
```

若遇到

```
lxc-start: tools/lxc_start.c: main: 366 The container failed to start.
lxc-start: tools/lxc_start.c: main: 368 To get more details, run the container in foreground mode.
lxc-start: tools/lxc_start.c: main: 370 Additional information can be obtained by setting the --logfile and --logpriority options.
```

代表沒有 start 起來，可以用

```shell
$ lxc-start -n hosta --logfile=logfile.log --logpriority=debug
```

來輸出 logfile，預設位址是在 ~/.local/share/lxc 下

## 問題與解法

### cgroup 權限問題

若 log 檔跑出

```
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/hugetlb/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/cpuset/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/memory/user.slice/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/blkio/user.slice/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/devices/user.slice/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied -  failed to create directory '/sys/fs/cgroup/pids/user.slice/user-1000.slice/lxc'
lxc-start 20180623060452.624 ERROR    lxc_utils - utils.c:mkdir_p:254 - Permission denied - failed to create directory '/sys/fs/cgroup/freezer/lxc'
```

可以在 /etc/group 下方的 lxc-dnsmasq 加入自己

### Network

若 log 檔跑出

```
lxc-start 20180623061551.112 ERROR    lxc_start - start.c:lxc_spawn:1222 - Failed to create the configured network.
```

記得在 /etc/lxc/lxc-usernet 加入網卡的設定

## LXC Commands

lxc start 成功後，就可以對它做一些操作

### 啟動 Container

```shell
$ lxc-attach -n hosta
```

### 關閉 Container

```shell
$ lxc-stop -n hosta
```

### 列出 Container

```shell
$ lxc-ls
```

### 複製 Container

```shell
$ lxc-copy -n hosta -N hostb
```

-n 接原本的 container 的名字，-N 接複製過去的 container 的名字