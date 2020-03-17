---
title: 'HTTP proxy (Squid)'
date: 2018-06-28 18:35:00
tags:
- NCTU
- NA
- squid
---

# HTTP proxy

環境：Ubuntu 16.04 (DigitalOcean)

## 安裝

```shell
$ sudo apt install squid
```

## Transparent proxy

添加一個群組叫 blocked_websites 並 deny 裡面設置的網站

/etc/squid/squid.conf

```
...
http_port 3128 transparent
...

# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
#http_access deny to_localhost

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
#http_access allow localnet
http_access allow localhost

acl blocked_websites dstdomain .msn.com .yahoo.com
http_access deny blocked_websites
http_access allow all

# And finally deny all other access to this proxy
# http_access deny all
...
```

最後記得把 deny all 註解掉，不然 host 會被斷網

最後在 iptables 內設定把從 80 port 進來的封包都通過 squid(3128)

```shell
$ sudo iptables -t nat -A PREROUTING -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 3128
```

## 測試

```shell
$ lxc-attach -n hostb
```

```shell
$ wget yahoo.com
```

應該會收到 403 Forbidden