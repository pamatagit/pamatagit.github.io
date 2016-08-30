---
layout: post
title: keystone中的默认domain
date: 2016-08-30
category: "keystone"
---

工作中遇到一个问题，用v3的api时能够认证成功，而用v2的api时就认证失败，经排查发现是默认domian引发的问题。

在keystone的v3 api中，user和project都有一个域的概念，也就是domain。user和project必须在某个domain中。不同domain中的user和project可以重名。所以在使用v3 api进行身份认证时，如果使用的是user_name和project_name，那么也必须提供user_domain和project_domain，这样才能确认其身份。

为了兼容v2 api,keystone有一个默认domain，该domain的id是default。在v2 api下创建的用户和项目也都属于该默认domain。

    +---------+---------+---------+--------------------+
    | ID      | Name    | Enabled | Description        |
    +---------+---------+---------+--------------------+
    | default | Default | True    | The default domain |
    +---------+---------+---------+--------------------+
    
    
而在使用v2 api进行身份认证时，由于v2中没有域的概念，如果提供的是user_name和project_name，那么keystone会去默认domain中去寻找该user和project。

如果使用用户名和项目名去进行身份认证，keystone会去default这个域中查找该用户和该项目，然后验证其密码和角色。

如果一个用户是在v3 api中创建，且创建在非默认domian中，要想使用该用户通过v2 api去进行身份认证时，必须使用该用户的user_id、project_id，而不能使用user_name和project_name，否则keystone会找不到该user_name，认证失败。

要想修改默认domain的id，可以在keystone的配置文件/etc/keystone/keystone.conf中配置：

    [identity]
    
    #
    # From keystone
    #
    
    # This references the domain to use for all Identity API v2 requests (which are
    # not aware of domains). A domain with this ID will be created for you by
    # keystone-manage db_sync in migration 008. The domain referenced by this ID
    # cannot be deleted on the v3 API, to prevent accidentally breaking the v2 API.
    # There is nothing special about this domain, other than the fact that it must
    # exist to order to maintain support for your v2 clients. (string value)
    #default_domain_id = default
    default_domain_id = new_domain_id



