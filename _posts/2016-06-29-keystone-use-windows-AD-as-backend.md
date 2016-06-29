---
layout: post
title: keystone对接windows AD
date: 2016-06-29
category: "keystone"
---

Keystone对接Windows AD实验
一、实验环境：
1.一个devstack环境
2.一个Windows AD环境，在AD中创建一个组织单元(OU)，在该组织单元中创建一个组(group)，在组中添加几个用户(user)。

![](http://i.imgur.com/CnEwFP4.png)

二、实验步骤：
1.配置keystone.conf使keystone支持为不同的域使用不同的认证后端：

![](http://i.imgur.com/US1GZOK.png)
 
每个域的配置文件放在/etc/keystone/domains目录下。

2.创建域并在/etc/keystone/domains目录下为该域创建keystone.DOMAIN_NAME.conf配置文件，本次实验中创建的域名为ldap。

  ![](http://i.imgur.com/JXFpym0.png)

配置keystone.ldap.conf：

![](http://i.imgur.com/K0GEWRD.png)

3.重启keystone服务，这时应该就可以查询到AD中的用户和组了
 
![](http://i.imgur.com/d3lh4d8.png)

4.要想使用AD中的用户来访问openstack，必须给这些用户或组分配角色。
 
![](http://i.imgur.com/5RB1UK9.png)

在admin这个项目中给group分配了admin的角色，这样这个group中的所有用户都有了该角色。

5.使用AD中的用户访问openstack
 
![](http://i.imgur.com/3VO7LhJ.png)