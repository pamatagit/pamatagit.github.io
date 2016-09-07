---
layout: post
title: fernet token
date: 2016-09-07
category: "keystone"
---

## 什么是fernet？##

fernet是keystone中除了uuid、pki/pkiz外的另一种形式的token，它使用共享的私钥去加密和解密token，所以它不需要保存在数据库中。

## fernet 相比 uuid、pki/pkiz 有哪些优势？ ##

快速：fernet最大的特色是不需要保存在数据库中，数据库中的token表一直为空，这样可以加快生成和验证token的速度。

轻量：另外pki/pkiz大小很容易超过1600字节，会超出HTTP头的大小限制，而fernet token大小一般在250字节以内，不会有这个问题。

## 怎么使用fernet？ ##

在keystone.conf中配置：

> [token]
> provider = keystone.token.providers.fernet.Provider
> 
> [fernet_token]
> key_repository = /etc/keystone/fernet-keys/
> max_active_keys = 3

然后执行：
> $ mkdir /etc/keystone/fernet-keys/
> $ keystone-manage fernet_setup

这样会在/etc/keystone/fernet-keys/目录下产生key文件。
有多个keystone节点时，需要将该目录下的key文件拷贝到其他节点，保证各个节点的key文件一致。

## fernet的key文件是什么？ ##

fernet的key文件有3种类型，primary key、secondary key、staged key。
加密token时使用primary key，解密token时会依次尝试使用这3种key去解密。

当通过keystone-manage fernet_rotate更新key文件时，staged key会变成primary key，并会生成一个新的staged key；primary key会变成secondary key，如果key文件数量超过了配置文件中的值，多余的secondary key会被删除，其他的保持不变。

每个key文件有256bit，前128bit用来签名，后128bit用来加密。

![](http://i.imgur.com/IXJot62.png)

## fernet token 内容 ##

![](http://i.imgur.com/Pv4wlf7.png)

## fernet token 产生过程 ##

![](http://i.imgur.com/v8lWnTu.png)

## fernet token 认证过程 ##

![](http://i.imgur.com/F8fUFKj.png)

## revocation evnet 对 fernet token 性能的影响 ##







