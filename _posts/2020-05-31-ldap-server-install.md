---
layout: post
title: "LDAP Serverのインストール"
date: 2020-05-31 12:00:00 +0300
description: 
img: ldap.jpg
---

今回はLDAPのサーバーをインストールしてみよう。  
まずはOSバージョンから確認する。
```
$ grep . /etc/*-release
/etc/centos-release:CentOS Linux release 7.5.1804 (Core) 
/etc/os-release:NAME="CentOS Linux"
/etc/os-release:VERSION="7 (Core)"
/etc/os-release:ID="centos"
/etc/os-release:ID_LIKE="rhel fedora"
/etc/os-release:VERSION_ID="7"
/etc/os-release:PRETTY_NAME="CentOS Linux 7 (Core)"
/etc/os-release:ANSI_COLOR="0;31"
/etc/os-release:CPE_NAME="cpe:/o:centos:centos:7"
/etc/os-release:HOME_URL="https://www.centos.org/"
/etc/os-release:BUG_REPORT_URL="https://bugs.centos.org/"
/etc/os-release:CENTOS_MANTISBT_PROJECT="CentOS-7"
/etc/os-release:CENTOS_MANTISBT_PROJECT_VERSION="7"
/etc/os-release:REDHAT_SUPPORT_PRODUCT="centos"
/etc/os-release:REDHAT_SUPPORT_PRODUCT_VERSION="7"
/etc/redhat-release:CentOS Linux release 7.5.1804 (Core) 
/etc/system-release:CentOS Linux release 7.5.1804 (Core) 
```

つぎは、openldapを含むパッケージをインストールする。
```
$ sudo yum -y install openldap compat-openldap openldap-clients openldap-servers openldap-servers-sql openldap-devel
```

インストールが終わったらsladp  (openldap)を実行し、常に実行するように登録する。
```
$ systemctl start slapd
$ systemctl enable slapd
```

ldapが正常に実行されたかポートを確認する。  
ldapのポートは389である。
```
$ netstat -antup | grep -i 389
(No info could be read for "-p": geteuid()=1000 but you should be root.)
tcp        0      0 0.0.0.0:389             0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::389                  :::*                    LISTEN      - 
```

ldap設定のために、まずldppasswordを作成する。  
slappasswdコマンドを実行すると、パスワードを聞いてそのパスワードを暗号化する。  
暗号化されたパスワードは設定に追加される。
```
$ slappasswd
New password: 
Re-enter new password: 
{SSHA}3ppRnFtC1srY+NRt2q13G2l7f74cqveA
```

ldap設定ファイルは/etc/openldap/slapd.d/にある。  
設定項目のうち、olcSuffixとolcRootDN、olcRootPWの変更が必要である。  
設定ファイルは/etc/openldap/slapd.d/cn=config/olcDatabase={2}hdb.ldifであるが、直接修正することはあまりおすすめしない。

まず、下のようにdb.ldifファイルを作成し、原本に適用する。
```
$ sudo vim db.ldif
```

db.ldifファイル内に以下の内容を追加する。  
olcRootPWには、以前生成した暗号化されたパスワードを入れる。dcの内容は使用するドメイン（ibm.com）を記載する。
```
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ibm,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=ldapadm,dc=ibm,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}3ppRnFtC1srY+NRt2q13G2l7f74cqveA
```

つぎは、作成したdb.ldifファイルの内容をldapserverにアップデートする。
```
$ sudo ldapmodify -Y EXTERNAL  -H ldapi:/// -f db.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"

modifying entry "olcDatabase={2}hdb,cn=config"
```

monitor接続アカウントの設定も必要なので、monitor.ldifファイルを作成し、オリジナルの設定ファイル(etc/openldap/slapd.d/cn=config/olcDatabase={1}monitor.ldif)にアップデートする。
```
$ sudo vim monitor.ldif
```

以下のようにldapadmアカウントのみ接続可能とする。
```
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external, cn=auth" read by dn.base="cn=ldapadm,dc=ibm,dc=com" read by * none
```

monitor.ldifファイルを保存して設定にアップデートする。
```
$ sudo ldapmodify -Y EXTERNAL  -H ldapi:/// -f monitor.ldif
SASL/EXTERNAL authentication started
SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
SASL SSF: 0
modifying entry "olcDatabase={1}monitor,cn=config"
```

次には、ldap databaseを設定する。  
データベース設定は、/usr/share/openldap-servers/のディレクトリ内にあるsampleファイルをアップデートして権限を与えればOK。
```
$ sudo cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
$ sudo chown ldap:ldap /var/lib/ldap/
```

ldap schemaを適用する。
```
$ sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
$ sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
$ sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

ドメイン情報も変更する。同様に、base.ldifファイルを作成してアップデートする。
```
$ sudo vim base.ldif
```

base.ldifファイル内に以下のように追加する。
```
dn: dc=ibm,dc=com
dc: ibm
objectClass: top
objectClass: domain

dn: cn=ldapadm,dc=ibm,dc=com
objectClass: organizationalRole
cn: ldapadm
description: LDAP Manager

dn: ou=People,dc=ibm,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=ibm,dc=com
objectClass: organizationalUnit
ou: Group
```

base.ldifファイルを作成したら、以下のコマンドでdirectory  structureをビルドする。
```
$ sudo ldapadd -x -W -D "cn=ldapadm,dc=ibm,dc=com" -f base.ldif
Enter LDAP Password: 
adding new entry "dc=ibm,dc=com"

adding new entry "cn=ldapadm,dc=ibm,dc=com"

adding new entry "ou=People,dc=ibm,dc=com"

adding new entry "ou=Group,dc=ibm,dc=com"
```

これでldap設定は完了。  
まず、例題で「chung」というアカウントを登録してみよう。  
今回もldifファイルを作成して設定にアップロードする。
```
$ sudo vim chung.ldif
```

chung.ldifファイルに以下のようにエントリを作成する。
```
dn: uid=chung,ou=People,dc=ibm,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: chung
uid: chung
uidNumber: 9999
gidNumber: 100
homeDirectory: /home/chung
loginShell: /bin/bash
gecos: chung [Admin (at) IBM]
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
```

以下のコマンドでアップデートしてユーザーを生成する。
```
$ sudo ldapadd -x -W -D "cn=ldapadm,dc=ibm,dc=com" -f chung.ldif
Enter LDAP Password: 
adding new entry "uid=chung,ou=People,dc=ibm,dc=com"
```

登録したchungアカウントに以下のコマンドでパスワードを設定する。  
- -s は設定しようとするパスワード
- -D はldapadmの情報
- -x はパスワードを設定しようとするユーザーの情報

```
$ sudo ldappasswd -s passw0rd -W -D "cn=ldapadm,dc=ibm,dc=com" -x "uid=chung,ou=People,dc=ibm,dc=com"
Enter LDAP Password: 
```

登録したユーザーの情報を検索するには、ldapsearchコマンドを使用するとよい。
```
$ sudo ldapsearch -x cn=chung -b dc=ibm,dc=com
# extended LDIF
#
# LDAPv3
# base  with scope subtree
# filter: cn=chung
# requesting: ALL
#

# chung, People, ibm.com
dn: uid=chung,ou=People,dc=ibm,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: chung
uid: chung
uidNumber: 9999
gidNumber: 100
homeDirectory: /home/chung
loginShell: /bin/bash
gecos: chung [Admin (at) IBM]
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
userPassword:: e1NTSEF9UjlwQWVJbzN0T3VEaDM2MmgzTW9xeEtpbzd3dlp3dXk=

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```