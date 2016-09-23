---
layout: post
title: keystone对接openldap
date: 2016-09-20
category: "keystone","ldap"
---

# 配置keystone实用openldap #

----------

## 安装ldap包 ##
> stack@devstack:~$ sudo pip install python-ldap ldappool

## 配置keystone.conf ##

Keystone默认使用sql作为后端驱动，可以调整配置文件使其支持为不同的域使用不同的后端驱动。
> stack@devstack:~$ vi /etc/keystone/keystone.conf
> [identity]
> domain_specific_drivers_enabled = true
> domain_config_dir = /etc/keystone/domains

## 创建使用openldap作为后端的域 ##
> stack@devstack:~$ openstack domain create myldap

## 为该域指定后端驱动 ##
域配置文件必须命名为keystone.DOMAIN_NAME.conf的格式。
> stack@devstack:~$ mkdir /etc/keystone/domains
> stack@devstack:~$ vi /etc/keystone/domains/keystone.myldap.conf
> [identity]
> #for kilo
> #driver = keystone.identity.backends.ldap.Identity
> #for mitaka 
> driver = ldap

> [ldap]
> url = ldap://10.89.153.167
> user = cn=Manager,dc=srv,dc=world
> password = 12345
> suffix = dc=srv,dc=world
> allow_subtree_delete = false
> query_scope = sub
> 
> user_tree_dn = ou=Users,dc=srv,dc=world
> user_objectclass = inetOrgPerson
> user_id_attribute = cn
> user_name_attribute = cn
> user_description_attribute = description
> user_mail_attribute = mail
> user_pass_attribute = userPassword
> user_enabled_attribute = enabled
> 
> group_tree_dn = ou=Groups,dc=srv,dc=world
> group_objectclass = groupOfNames
> group_id_attribute = cn
> group_name_attribute = cn
> group_desc_attribute = description
> group_member_attribute = member
> 
> user_allow_create = false
> user_allow_update = false
> user_allow_delete = false
> group_allow_create = false
> group_allow_update = false
> group_allow_delete = false

## 重启keystone服务 ##
> stack@devstack:~$ sudo service apache2 restart
> stack@devstack:~$ openstack user list --domain myldap
> +---------------------------------------------------------------------------------------------------+------+
> | ID                                                            | Name 
> +---------------------------------------------------------------------------------------------------+------+
> | f6dd589e4dc1b859080bf3d34735703d0d0e4f3b42d57c8726d70cdb0077e1a6 | gx 
> +---------------------------------------------------------------------------------------------------+------+
> 
> stack@devstack:~$ openstack group list --domain myldap
> +---------------------------------------------------------------------------------------------------+------+
> | ID                                                            | Name 
> +---------------------------------------------------------------------------------------------------+------+
> | 4ec2c995331324c9c1728b5cbc6b441cfc5db42f6cabd7beb69b6951187100a5 | nova 
> +---------------------------------------------------------------------------------------------------+------+
> 
> stack@devstack:~$ openstack user list --group 4ec2c995331324c9c1728b5cbc6b441cfc5db42f6cabd7beb69b6951187100a5
> +---------------------------------------------------------------------------------------------------+------+
> | ID                                                            | Name 
> +---------------------------------------------------------------------------------------------------+------+
> | f6dd589e4dc1b859080bf3d34735703d0d0e4f3b42d57c8726d70cdb0077e1a6 | gx   
> 
+---------------------------------------------------------------------------------------------------+------+

## 创建project并给用户绑定角色 ##
> stack@devstack:~$ openstack project create myproject --domain myldap
> +------------------+----------------------------------------------------+
> | Field       | Value                            
> +------------------+---------------------------------------------------+
> | description  |                                 
> | domain_id   | 56f5f748ca984dad90c4bd1401ad1aaa  
> | enabled     | True                             
> | id          | 9bd17a50ec9042448d0efa309db46e67  
> | name       | myproject                        
> +------------------+---------------------------------------------------+
> stack@devstack:~$ openstack role add --project 9bd17a50ec9042448d0efa309db46e67     --group 4ec2c995331324c9c1728b5cbc6b441cfc5db42f6cabd7beb69b6951187100a5 admin

## 使用openldap中的用户访问openstack ##
> stack@devstack:~$ vi myldap-openrc.sh
> export OS_AUTH_URL="http://192.168.122.12:5000/v3"
> export OS_USER_DOMAIN_ID="56f5f748ca984dad90c4bd1401ad1aaa"
> export OS_PROJECT_DOMAIN_ID="56f5f748ca984dad90c4bd1401ad1aaa"
> export OS_PROJECT_NAME="myproject"
> export OS_USERNAME="gx"
> export OS_PASSWORD= "x"
> export OS_VOLUME_API_VERSION=2
> export OS_IDENTITY_API_VERSION=3
> 
> stack@devstack:~$ source myldap-openrc.sh
> stack@devstack:~$ nova net-list

# keystone.DOMAIN_NAME.conf配置项说明 #

----------

## 常用配置项 ##
> [identity]
> #for kilo
> #driver = keystone.identity.backends.ldap.Identity
> #for mitaka 
> driver = ldap

该配置项告诉keystone使用LDAP作为[identity]后端驱动(默认为SQL)。

在[ldap]中，添加了LDAP专用设置：

> url = ldap://10.89.153.167

指定了ldap服务器的URL。

> user = cn=Manager,dc=srv,dc=world
> password = 12345

指定了ldap服务器管理员的dn和密码。会使用该用户去登陆ldap并读取ldap中数据。

> suffix = dc=srv,dc=world

当不指定user_tree_dn和group_tree_dn时，会通过在该配置项的值前面添加”ou=Users”和”ou=UserGroups”来构成user_tree_dn和group_tree_dn。

> query_scope = sub

指定了ldap的查找深度。值为”one”时，只在user_tree和group_tree的顶层子树中查找；值为”sub”时，会在所有子树下去查找。

> user_tree_dn = ou=Users,dc=srv,dc=world

指定了用户树。查找用户时去该树下查找。

> user_objectclass = inetOrgPerson

指定了用户的ldap对象类型。查找用户时只查找该类型的对象，忽略其他任意类型的对象。

> user_id_attribute = cn
> user_name_attribute = cn
> user_description_attribute = description
> user_mail_attribute = mail
> user_pass_attribute = userPassword

将ldap中的用户属性映射到keystone中。user_id和user_name对应的属性必须是能唯一标识该用户的，且前者长度不能超过个64字符。

> group_tree_dn = ou=Groups,dc=srv,dc=world

指定了group树。查找group时去该树下查找。

> group_objectclass = groupOfNames

指定了group的ldap对象类型。查找group时只查找该类型的对象，忽略其他任意类型的对象。

> group_id_attribute = cn
> group_name_attribute = cn
> group_desc_attribute = description
> group_member_attribute = member

将ldap中的group属性映射到keystone中。member属性代表该group中有哪些用户。

> user_allow_create = false
> user_allow_update = false
> user_allow_delete = false
> group_allow_create = false
> group_allow_update = false
> group_allow_delete = false

指定了keystone是否可以对ldap做写操作（默认为true）。这些配置项在M版已经被废弃了，N版中将不再支持这些配置，默认ldap对keystone是只读的。

## 其他参数 ##

> use_dumb_member = false
> dumb_member = “cn=dumb,dc=nonexistent”

dumb_member只有在group是可写的，而且group class需要定义一个member属性时才会被用到。usb_dumb_member为true时，若创建一个新的group，则group的member属性会设置成dumb_member。

> user_filter = (employeeType=1)(title=123)
> group_filter = (o=aa)

会分别与user_objectclass和group_objectclass组成查询过滤条件，返回满足条件的对象。

> user_enabled_attribute = enabled

表示用户的enabled属性所对应的ldap servier中的用户enabled属性。在WindowsAD中表示用户是否enable的属性为”userAccountControl”，且该属性不是一个bool值，而是一个int型数字。


> user_enabled_default = True

表示用户enabled的默认值。在”user_enabled_attribute = userAccountControl” （WindowsAD中表示用户enabled的属性）时该值一般设为512。


> user_enabled_invert = false

有些ldap server中用一个lock属性来表示用户是否enable。lock属性为true表示用户为disabled，为false表示用户为enabled。这时候就需要用到”user_enabled_invert = true”。当使用”user_enabled_mask”和”user_enabled_emulation”时该配置项无效。


> user_enabled_mask = 0

有些ldap server表示用户enabled属性时不是用的true或者false，而是用一个int型数字表示。user_enabled_mask指定了表示enabled属性的二进制位数，设定为0表示不使用。
一般在”user_enabled_attribute = userAccountControl”时设置为”user_enabled_mask = 2”。

> user_enabled_emulation = true
> user_enabled_emulation_dn = “ou=enabledusers,o=acme.com”
> user_enabled_emulation_user_group_config = false

Openldap中没有表示用户enable的属性，在”user_enabled_emulation = true”时，keystone将在”user_enabled_emulation_dn”这个group中的用户视为enabled，不在该group中的用户视为disabled。”user_enabled_emulation_user_group_config”为true时，通过核对group_member_attribute属性中是否有该用户来决定用户是否在group中。

> allow_subtree_delete = false

openldap允许直接删除包含子树的树，将”allow_subtree_delete”设为true可以通过keystone删除包含子树的user_tree和group_tree。

> page_size = 0

page_size设为非0值时，若ldap支持page_size，ldap每次向keystone发送的entry数据为page_size值。但是keystone api一次性返回所有的entry数据，因此该配置项并不影响keystone api层面，只影响ldap和keystone底层的交互。

> alias_dereferencing = default

配置ldap的别名解引用设置。default表示使用ldap server的配置，其他可选值有never、always、finding、searching。never表示从不dereference，always表示一直dereference，finding表示只对指定的对象dereference而对其下属对象不dereference，searching表示对指定的对象不dereference而对其下属对象dereference。

> debug_level = 0

该值非0时，表示会debug到ldap层，数值代表了debug的深度，具体要参考ldap文档。

> user_attribute_ignore = default_project_id

该配置项指定的是keystone用户不需要从ldap映射的属性。很少有ldap服务有相应的属性，因此该配置项默认值是default_project_id。

> user_default_project_id_attribute = None

指定了映射到用户默认projecti d的ldap属性。该配置项很少用到。

> user_additional_attribute_mapping = sn:description

keystone在创建用户时将keystone中的属性映射到ldap中。若 ldap中的用户必须有cn，sn，uid这3种属性，而keystone中将useriduid,usernamecn，sn属性不能指定，配置了该配置项后，可以将descripiton指定成sn属性。

> group_attribute_ignore = 
> group_additional_attribute_mapping = 

与”user_attribute_ignore”和”user_additional_attribute_mapping”类似。

> group_members_are_ids = false

如果group中的members是user id而不是dn，将该配置项设为true。

> use_pool = false
> pool_size = 10
> pool_retry_max = 3
> pool_retry_delay = 0.1
> pool_connection_timeout = -1
> pool_connection_lifetime = 600

use_pool默认为false，设为true时keystone会使用ldappool来优化性能。该pool不处理用户认证的请求。

> use_auth_pool = false
> auth_pool_size = 100
> auth_pool_connection_lifetime = 60

use_auth_pool设为true时可以使用一个专门的pool来进行用户认证，可以进一步优化性能。

> use_tls = True
> tls_cacertfile = /etc/keystone/ssl/certs/cacert.pem
> tls_cacertdir = /etc/keystone/ssl/certs/
> tls_req_cert = demand

当use_tls为true时，keystone会通过ssl方式访问ldap。Tls_cacertfile指定了certificate files所在位置，若有多个文件，需要全部列出来。Tls_cacertdir指定了certifications所在目录。Tls_req_cert可以有3个值，demand、never、allow。默认值demand表示keyistone一直会验证certificate，若验证失败则断开连接；never表示从不验证certificate；allow表示如果没有提供certificate则不验证，若提供了则验证，验证失败时会断开连接。
