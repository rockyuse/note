---
title: 'MTA (postfix)、MRA (dovecot)'
tags:
- NCTU
- Mail
- NA
---

# MTA、MRA

環境：Ubuntu 16.04 (DigitalOcean)

Mail Server 相關知識可以參考[鳥哥](http://linux.vbird.org/linux_server/0380mail.php)

筆記一下幾個名詞：

>**MUA** (Mail User Agent)：郵件使用者代理人，如 Thunderbird, Outlook... MUA 主要的功能就是收受郵件主機的電子郵件，以及提供使用者瀏覽與編寫郵件的功能
>
>**MTA** (Mail Transfer Agent)：郵件傳送代理人
>>收受信件：使用 **SMTP** (port 25)
>>轉遞信件：若目的地不是本身的用戶，則 MTA 會將該信傳送給下一部主機
>
>**MDA** (Mail Delivery Agent)：郵件遞送代理人，掛在 MTA 下的小程式，用來分析由 MTA 所收到的信件表頭或內容等資料， 來決定這封郵件的去向。
>
>**MRA** (Mail Retrieval Agent)：使用者可以透過 MRA 伺服器提供的郵政服務協定收受信件
>>**POP3** (Post Office Protocol)
>>* MUA 透過 POP3 連接到 MRA 的 port 110
>>* MRA 確認該使用者帳號/密碼沒有問題後，會前往該使用者的 Mailbox 取得使用者的信件並傳送給使用者的 MUA
>>* 當所有的信件傳送完畢後，使用者的 mailbox 內的資料將會被刪除
>
>>**IMAP** (Internet Message Access Protocol)
>>* 讓你將 mailbox 的資料轉存到你主機上的家目錄，亦即 /home/帳號/ 的目錄下

## MTA：postfix

### 安裝

```shell
$ sudo apt install postfix
```

### 基本設定

/etc/postfix/main.cf

```
...
myhostname = kaiiiz.nctucs.net
...
mydestination = $myhostname, kaiiiz.nctucs.net, na-server-2, localhost.localdomain, localhost
...
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [:～:1]/128
...
```

> mydestination：The mydestination parameter specifies what domains this machine will deliver locally, instead of forwarding to another machine.
> mynetworks：設定可以幫忙relay的位址，設定這個值會取代 mynetworks_style 的參數

### 測試

```shell
$ telnet localhost 25
```

## MRA：dovecot

### 安裝

```shell
$ sudo apt-get install dovecot-imapd dovecot-pop3d
```

### 基本設定

config 都放在 /etc/dovecot/conf.d 這個目錄下，裝完不用做設定就能用了

### 測試

```shell
$ netstat -tlnp | grep dovecot
tcp        0      0 0.0.0.0:110             0.0.0.0:*               LISTEN      1634/dovecot
tcp        0      0 0.0.0.0:143             0.0.0.0:*               LISTEN      1634/dovecot
tcp6       0      0 :::110                  :::*                    LISTEN      1634/dovecot
tcp6       0      0 :::143                  :::*                    LISTEN      1634/dovecot
```

> -t：tcp
> -l：Show only listening sockets.
> -n：Show numerical addresses instead of trying to determine symbolic host, port or user names.
> -p：Show the PID and name of the program to which each socket belongs.

## 常用指令

### list queue

```shell
$ postqueue -p
```

或是

```shell
$ mailq
```

### flush queue

```shell
$ postqueue -f
```

### clean queue

```shell
$ postsuper -d ALL
```

## 後記

備份一些檔案，可以當作參考

* [postfix](https://gist.github.com/3f4505f446c5c24b7db4fe26f39f38f8.git)
* [dovecot](https://gist.github.com/536e0a6502aa7c9a5ed51fdef382bd46.git)