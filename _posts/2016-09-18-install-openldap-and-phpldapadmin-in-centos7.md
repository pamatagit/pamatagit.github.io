---
layout: post
title: 在centos7中安装openldap和phpldapadmin
date: 2016-09-18
category: "ldap"
---
# 安装和配置openldap #

----------

## 安装openldap server ##
> [root@dlp ~]# yum -y install openldap-servers openldap-clients
> 
> [root@dlp ~]# cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG 
> 
> [root@dlp ~]# chown ldap. /var/lib/ldap/DB_CONFIG 
> 
> [root@dlp ~]# systemctl start slapd 
> 
> [root@dlp ~]# systemctl enable slapd 

## 设置openldap管理密码 ##

> [root@dlp ~]# slappasswd 
> 
> New password:
> Re-enter new password:
> {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
> 
> [root@dlp ~]# vi chrootpw.ldif
> 
>  dn: olcDatabase={0}config,cn=config
> changetype: modify
> add: olcRootPW
> olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
> 
> [root@dlp ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif 
> 
> SASL/EXTERNAL authentication started
> SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
> SASL SSF: 0
> modifying entry "olcDatabase={0}config,cn=config"

## 导入基础架构 ##

> [root@dlp ~]# ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
> 
> SASL/EXTERNAL authentication started
> SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
> SASL SSF: 0
> adding new entry "cn=cosine,cn=schema,cn=config"
> 
> [root@dlp ~]#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
>  
> SASL/EXTERNAL authentication started
> SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
> SASL SSF: 0
> adding new entry "cn=nis,cn=schema,cn=config"
> 
> [root@dlp ~]#ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif 
> 
> SASL/EXTERNAL authentication started
> SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
> SASL SSF: 0
> adding new entry "cn=inetorgperson,cn=schema,cn=config"

## 创建域 ##

### 生成openldap管理员密码 ###

> [root@dlp ~]# slappasswd 
> 
> New password:
> Re-enter new password:
> {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx

### 创建openldap域管理员 ###
> [root@dlp ~]# vi createdomain.ldif
> 
> dn: olcDatabase={1}monitor,cn=config
> changetype: modify
> replace: olcAccess
> olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by  dn.base="cn=Manager,dc=srv,dc=world" read by * none
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> replace: olcSuffix
> olcSuffix: dc=srv,dc=world
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> replace: olcRootDN
> olcRootDN: cn=Manager,dc=srv,dc=world
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> add: olcRootPW
> olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
> 
> dn: olcDatabase={2}hdb,cn=config
> changetype: modify
> add: olcAccess
> olcAccess: {0}to attrs=userPassword,shadowLastChange by  dn="cn=Manager,dc=srv,dc=world" write by anonymous auth by self write by * none
> olcAccess: {1}to dn.base="" by * read
> olcAccess: {2}to * by dn="cn=Manager,dc=srv,dc=world" write by * read

> [root@dlp ~]# ldapmodify -Y EXTERNAL -H ldapi:/// -f chdomain.ldif 
> 
> SASL/EXTERNAL authentication started
> SASL username: gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth
> SASL SSF: 0
> modifying entry "olcDatabase={1}monitor,cn=config"
> modifying entry "olcDatabase={2}hdb,cn=config"
> modifying entry "olcDatabase={2}hdb,cn=config"
> modifying entry "olcDatabase={2}hdb,cn=config"

### 创建域及组织单元 ###
> [root@dlp ~]# vi basedomain.ldif
> 
> dn: dc=srv,dc=world
> objectClass: top
> objectClass: dcObject
> objectclass: organization
> o: Server World
> dc: Srv
> 
> dn: ou=Users,dc=srv,dc=world
> objectClass: organizationalUnit
> ou: Users
> 
> dn: ou=Groups,dc=srv,dc=world
> objectClass: organizationalUnit
> ou: Groups
> 
> [root@dlp ~]#ldapadd -x -D cn=Manager,dc=srv,dc=world -W -f basedomain.ldif 
> 
> Enter LDAP Password:
> adding new entry "dc=srv,dc=world"
> adding new entry "ou=People,dc=srv,dc=world"
> adding new entry "ou=Group,dc=srv,dc=world"

### 验证 ###
> [root@dlp ~]#ldapsearch -x –W -D cn=Manager,dc=srv,dc=world –b dc=srv,dc=world
> 
> Enter LDAP Password:
> dn: dc=srv,dc=world
> objectClass: top
> objectClass: dcObject
> objectClass: organization
> o: Server World
> dc: Srv
> 
> dn: ou=Users,dc=srv,dc=world
> objectClass: organizationalUnit
> ou: Users
> 
> dn: ou=Groups,dc=srv,dc=world
> objectClass: organizationalUnit
> ou: Groups

## 打开防火墙 ##
> [root@dlp ~]#firewall-cmd --add-service=ldap --permanent
> success
> 
> [root@dlp ~]# firewall-cmd --reload
> Success

# 安装和配置phpldapadmin #

----------

## 添加epel源 ##
> [root@dlp ~]#vi /etc/yum.repos.d/epel.repo
> 
> [epel]
> name=epel
> baseurl=http://ftp.jaist.ac.jp/pub/Linux/Fedora/epel//7/x86_64/
> gpgcheck=0
> priority=1
> 
> [root@dlp ~]# yum clean all
> 
> [root@dlp ~]# yum makecache

## 安装phpldapadmin ##
> [root@dlp ~]# yum --enablerepo=epel -y install phpldapadmin
> 
> [root@dlp ~]# vi /etc/phpldapadmin/config.php
> 
> $servers->setValue('login','attr','dn');  # line 397: uncomment, line 398: comment out
> // $servers->setValue('login','attr','uid');
> 
> [root@dlp ~]# vi /etc/httpd/conf.d/phpldapadmin.conf
> 
> Require all granted  > # line 12: add access permission
> 
> [root@dlp ~]# systemctl restart httpd 

## 设置允许通过httpd访问ldap ##
> [root@dlp ~]# getsebool httpd_can_connect_ldap
> 
> httpd_can_connect_ldap -->off
> 
> [root@dlp ~]# setsebool -P httpd_can_connect_ldap on
> 
> [root@dlp ~]# getsebool httpd_can_connect_ldap
> 
> httpd_can_connect_ldap -->on
