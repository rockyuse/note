---
title: 'FTP Server with LDAP-Backend'
date: 2018-06-28 14:35:00
tags:
- NCTU
- NA
- proftpd
- openldap
---

# FTP Server with LDAP-Backend

建立一個以 LDAP 管理用戶，且只有 LDAP 中特定的 group 能登錄的 FTP Server。

環境：Ubuntu 16.04 (DigitalOcean)

## OpenLDAP

### 安裝

```shell
$ sudo apt-get install slapd ldap-utils
```

### Config

```shell
$ sudo dpkg-reconfigure slapd
```

### 新增 OU

添加 People 和 Groups 兩個 ou

add_nodes.ldif

```
dn: ou=people,dc=kaiiiz,dc=nctucs,dc=net
objectClass: organizationalUnit
ou: People

dn: ou=groups,dc=kaiiiz,dc=nctucs,dc=net
objectClass: organizationalUnit
ou: Groups
```

之後加入 LDAP 中

```shell
$ sudo ldapadd -x -W -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f add_nodes.ldif
```

> -x: Use simple authentication instead of SASL.
> 
> -W: Prompt for simple authentication.
> 
> -D **binddn**: Use the Distinguished Name **binddn** to bind to the LDAP directory.For SASL binds, the server is expected to ignore this value.
> 
> -f **file**: Read the entry modification information from **file** instead of from standard input.

### LDAP 資料庫

檔案存在 /etc/openldap/slapd.d/cn=config

```shell
$ sudo ls /etc/openldap/slapd.d/cn=config
cn=module{0}.ldif  cn=schema  cn=schema.ldif  olcBackend={0}mdb.ldif  olcDatabase={0}config.ldif  olcDatabase={-1}frontend.ldif  olcDatabase={1}mdb.ldif
```

要注意的是下面兩條：

* cn=module{0}.ldif
* olcDatabase={1}mdb.ldif

### memberOf

參考資料：[在openLdap上新增memberOf屬性](https://tw.saowen.com/a/51575ba6787a4141906bbedc777549af7dc4bb19044d67c4e06a8105184f2100)

有幾個新舊版的差異：

* 使用者組的objectClass從 **groupOfNames** 變成了 **groupOfUniqueNames**
* 組裡邊的使用者的屬性的名稱從 **member** 變成了 **uniqueMember**

#### 新增模組

memberof_config.ldif

```
dn: cn=module,cn=config
cn: module
objectClass: olcModuleList
olcModuleLoad: memberof
olcModulePath: /usr/lib/ldap

dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfUniqueNames
olcMemberOfMemberAD: uniqueMember
olcMemberOfMemberOfAD: memberOf
```

加進 LDAP 中

```shell
$ sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f memberof_config.ldif
```

> -Q: Enable SASL Quiet mode.  Never prompt.
> 
> -Y **mech**: Specify the SASL mechanism to be used for authentication. If it's not specified, the program will  choose  the  best  mechanism  the  server knows.
> 
> -H **ldapuri**: Specify  URI(s)  referring  to the ldap server(s); only the protocol/host/port fields are allowed; a list of URI, separated by whitespace or commas is expected.

檢查是否有多一個模組

```shell
$ sudo ls /etc/openldap/slapd.d/cn=config
cn=module{0}.ldif  cn=schema       olcBackend={0}mdb.ldif      olcDatabase={-1}frontend.ldif  olcDatabase={1}mdb.ldif
cn=module{1}.ldif  cn=schema.ldif  olcDatabase={0}config.ldif  olcDatabase={1}mdb
```

確認 {0} 和 {1} 哪個模組是 memberOf 的

```shell
$ sudo cat cn=config/cn=module{1}.ldif
# AUTO-GENERATED FILE - DO NOT EDIT!! Use ldapmodify.
# CRC32 3b859294
dn: cn=module{1}
objectClass: olcModuleList
cn: module{1}
olcModulePath: /usr/lib/ldap
olcModuleLoad: {0}memberof
structuralObjectClass: olcModuleList
entryUUID: 54d20f40-0ef2-1038-95a9-c9d65903e937
creatorsName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
createTimestamp: 20180628074053Z
entryCSN: 20180628074053.872377Z#000000#000#000000
modifiersName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
modifyTimestamp: 20180628074053Z
```

確認 cn=module{1} 這個模組是 memberOf 的模組

#### Load refint

refint1.ldif

```
dn: cn=module{1},cn=config
add: olcmoduleload
olcmoduleload: refint
```

載入 refint 模組

```shell
$ sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif
```

比較一下差異：

Before:

```
...
olcModulePath: /usr/lib/ldap
olcModuleLoad: {0}memberof
structuralObjectClass: olcModuleList
entryUUID: 54d20f40-0ef2-1038-95a9-c9d65903e937
creatorsName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
...
```

After:

```
...
olcModulePath: /usr/lib/ldap
olcModuleLoad: {0}memberof
olcModuleLoad: {1}refint
structuralObjectClass: olcModuleList
entryUUID: 54d20f40-0ef2-1038-95a9-c9d65903e937
creatorsName: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
...
```

中間多了一個 olcModuleLoad: {1}refint

#### Configure refint

refint2.ldif

```
dn: olcOverlay={1}refint,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: {1}refint
olcRefintAttribute: memberof uniqueMember manager owner
```

{1}refint 要寫對，最後新增

```shell
$ sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif
```

### 新增 member

add_member.ldif

```
dn: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
cn: Andy Zheng
gn: Andy
sn: Lee
uid: andy
uidNumber: 5001
gidNumber: 10000
homeDirectory: /home/andy
mail: andy@kaiiiz.nctucs.net
objectClass: top
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
loginShell: /bin/bash
userPassword: {SHA}YOURPASSWORD
```

> gn: givenName
> sn: surname

生成加密後的密碼的指令：

```shell
$ slappasswd -h {SHA} -s your_password
```

把 member 加入 LDAP

```shell
$ sudo ldapadd -x -W -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f add_member.ldif
```

測試一下

```shell
$ sudo ldapsearch -x -LLL -H ldap:/// -b uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net dn
dn: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
```

> -LLL: Search results are display in LDAP Data Interchange Format detailed in ldif(5). A single -L restricts the output to **LDIFv1**. A second -L **disables comments**.  A third -L **disables printing of the LDIF version**.  The default is to use an extended version of LDIF
> 
> -b **searchbase**: Use **searchbase** as the starting point for the search instead of the default.

### 新增 group

sysgroup.ldif

```
dn: cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net
objectClass: groupOfUniqueNames
cn: sysgroup
description: All users
uniqueMember: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
```

把 group 加入 LDAP

```shell
$ sudo ldapadd -x -W -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f sysgroup.ldif
```

測試一下

```shell
$ sudo ldapsearch -x -LLL -H ldap:/// -b cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net dn
dn: cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net
```

### 把 member 加入 group 中

add_member_to_group.ldif

```
dn: cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net
changetype: modify
add: uniqueMember
uniqueMember: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
```

加進 LDAP

```shell
$ sudo ldapmodify -x -W -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f add_member_to_group.ldif
```

測試一下

```shell
$ sudo ldapsearch -x -LLL -H ldap:/// -b uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net dn memberof
dn: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
memberOf: cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net
```

### 刪除 entry

delete.ldif

```
dn: cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net
changetype: delete
```

用 ldapmodify 修改

```shell
$ ldapmodify -x -W -H ldap:/// -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f delete.ldif
```

### 取代 entry

replace.ldif

```
dn: uid=andy,ou=people,dc=kaiiiz,dc=nctucs,dc=net
changetype: modify
replace: userPassword
userPassword: {SHA}YOURPASSWORD
```

用 ldapmodify 修改

```shell
$ ldapmodify -x -W -H ldap:/// -D cn=admin,dc=kaiiiz,dc=nctucs,dc=net -f replace.ldif
```

## FTP + LDAP

先進入 hostb

```shell
$ lxc-attach -n hostb
```

### 安裝

```shell
$ sudo apt install proftpd proftpd-mod-ldap
```

### standalone v.s. inetd

standalone：FTP 會常駐於記憶體，優點是反應快
inetd：當外部連線發送請求時才調用 FTP，優點是不佔用系統資源

### 設定 proftpd

/etc/proftpd/proftpd.conf

```
Server "kaiiiz.nctucs.net"
...
DefaultRoot ~
...
RequireValidShell off
...
Include /etc/proftpd/ldap.conf
```

### 開啟 LDAP 的模組

/etc/proftpd/modules.conf

```
LoadModule mod_ldap.c
```

### 接上 Server 的 LDAP

/etc/proftpd/ldap.conf

```
<IfModule mod_ldap.c>

LDAPServer ldap://10.0.3.1
LDAPBindDN "cn=admin,dc=kaiiiz,dc=nctucs,dc=net" "password"
LDAPUsers "uid=%u,ou=people,dc=kaiiiz,dc=nctucs,dc=net" "memberOf=cn=sysgroup,ou=groups,dc=kaiiiz,dc=nctucs,dc=net"

LDAPDefaultUID 107
LDAPDefaultGID 65534

LDAPGenerateHomedir on
LDAPGenerateHomedirPrefix /home/ftp
LDAPForceGeneratedHomedir on
LDAPGenerateHomedirPrefixNoUsername off
CreateHome on

</IfModule>
```

LDAPDefaultUID 及 LDAPDefaultGID 可由下方這條指令得到：

```shell
$ cat /etc/passwd | grep proftpd
```

> LDAPUsers: 要 memberOf sysgroup 的成員才能 Login FTP
> 
> LDAPGenerateHomedir, LDAPGenerateHomedirPrefix, LDAPForceGeneratedHomedir: 設定家目錄為 /home/ftp ，作為 LDAP 用戶登入的家目錄
> 
> LDAPGenerateHomedirPrefixNoUsername: 在家目錄下，建立用戶的個人目錄，若設為 on 則所有人共享同個目錄
> 
> CreateHome: 若家目錄不存在即建立它

### 測試

在 Server 中

```shell
$ ftp 10.0.3.150
Connected to 10.0.3.150.
220 ProFTPD 1.3.5a Server (kaiiiz.nctucs.net) [::ffff:10.0.3.150]
Name (10.0.3.150:andy): andy
331 Password required for andy
Password:
230 User andy logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```