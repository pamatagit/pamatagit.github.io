---
layout: post
title: token的生成与验证
date: 2016-08-26
category: "keystone"
---

以nova flavor-list命令为例，整理nova与keystone的交互。

1.novaclient读取环境变量中的身份信息，向keystone发送请求：
`POST http://keystone:5000/v3/auth/tokens`

2.keystone认证身份信息，返回novaclient一个token，假设为token1

3.novaclient带着token1向nova-api发送请求：
`GET http://nova:8774/flavors/detail  "X-Auth-Token: token1"`

4.nova-api收到请求后，读取配置文件中keystone_auth的身份信息，向keystone发送请求：
`POST http://keystone:5000/v3/auth/tokens`

5.keystone认证身份信息，返回nova-api一个token，假设为token2

6.nova-api收到请求后向keystone验证token1，若token1为uuid或fernet，发送：
`GET http://keystone:5000/v3/auth/tokens "X-Subject-Token: token1" "X-Auth-Token: token2"`；
若token1为pki/pkiz则发送:
`GET http://keystone:5000/v2.0/tokens/revoked`

7.keystone收到请求，验证token1，返回验证成功，执行第9步；或返回revocation list，执行第8步。。

8.若pkiz不在revocation list中，用cms解码pkiz成功且未过期，验证成功，执行第9步。

9.nova-api执行查询flavor的操作



# novaclient向keystone请求token #

---------------------------------
novaclient解析命令行参数后，会构造一个client。构造client时会传一个session和一个keystone_auth进去，session用来发送请求，auth_plugin用来认证身份信息。
{% highlight python %}

    keystone_session = ksession.Session.load_from_cli_options(args)
    keystone_auth = self._get_keystone_auth(**kwargs)
{% endhighlight %}
这里的keystone_auth是keystoneclient.auth.identity.generic.password.PassWord实例，包含了身份认证需要的信息，如username、password、project、domain等。

novaclient向nova-api发送请求之前，会先通过session，将keystone_auth信息发送到keystone，获取一个token。
获取token的动作是通过keystone_auth的get_auth_ref方法完成的。
{% highlight python %}

    def get_auth_ref(self, session, **kwargs):
        if not self._plugin:
            self._plugin = self._do_create_plugin(session)
        return self._plugin.get_auth_ref(session, **kwargs)
{% endhighlight %}
该方法会先创建一个plugin，根据api版本，选择创建v2.Password还是v3.Password，然后调用plugin的get_auth_ref。

本文以v3.Password为例，其get_auth_ref会将身份信息组合成:
`{"auth": {"scope": {"project": {"domain": {"id": "default"}, "name": "admin"}}, "identity": {"password": {"user": {"domain": {"id": "default"}, "password": "x", "name": "admin"}}, "methods": ["password"]}}}`，并发送到keystone，keystone会返回一个token。

获取到token后，从token中提取endpoint，将请求发送到endpoint。

# keystone认证身份信息并生成一个token #

----------
认证身份信息会去identity的backends（sql、ldap）里认证。
> [identity]
> driver = keystone.identity.backends.sql.Identity

认证方法有password、token、oauth1、external这4种，根据用户提供的身份信息选择其中一种。
> [auth]
> methods = external,password,token,oauth1

若backend用sql，且提供的是password，则会去sql中查询用户的用户名、密码等，并与请求中提供的用户信息比对，若一致则通过认证。

身份认证通过后，会调用token_provider_api去生成一个token，这里的provider有uuid、pkiz/pki、fernet这几种。

> [token]
> driver = keystone.token.persistence.backends.sql.Token
> provider = keystone.token.providers.[fernet|pkiz|pki|uuid].Provider

对于uuid、pki/pkiz，token_data的内容有scope、user、role、catalog、date等：
{% highlight python %}

        self._populate_scope(token_data, domain_id, project_id)
        self._populate_user(token_data, user_id, trust)
        self._populate_roles(token_data, user_id, domain_id, project_id, trust,
                             access_token)
        self._populate_audit_info(token_data, audit_info)

        if include_catalog:
            self._populate_service_catalog(token_data, user_id, domain_id,
                                           project_id, trust)
        self._populate_service_providers(token_data)
        self._populate_token_dates(token_data, expires=expires, trust=trust,
                                   issued_at=issued_at)
        self._populate_oauth_section(token_data, access_token)
{% endhighlight %}
token_id也保存在token_data中，然后会在数据库中创建token记录，记录包含token_data。

对于fernet，token_data的内容有：
{% highlight python %}

        token_data = self.v3_token_data_helper.get_token_data(
            user_id,
            method_names,
            auth_context.get('extras') if auth_context else None,
            domain_id=domain_id,
            project_id=project_id,
            expires=expires_at,
            trust=trust,
            bind=auth_context.get('bind') if auth_context else None,
            token=token_ref,
            include_catalog=include_catalog,
            audit_info=parent_audit_id)
{% endhighlight %}
而token_id是将部分token_data组合之后用密钥加密签名后生成的。
{% highlight python %}

    payload = ProjectScopedPayload.assemble(
    	user_id,
    	methods,
    	project_id,
    	expires_at,
    	audit_ids)
    
    versioned_payload = (version,) + payload
    serialized_payload = msgpack.packb(versioned_payload)
    token = self.pack(serialized_payload)
{% endhighlight %}
uuid的token_id只是一个随机生成的字串，不包含任何用户信息。
而pki/pkiz和fernet的token_id都是将用户信息通过某种方式加密后产生的。


# nova-api收到请求后验证请求中附带的token #

----------
nova-api收到请求后会通过keystonemiddleware插件来验证请求中附带的token1。

若token1为pki/pkiz，会先去keystone获取一个新的token2，然后用token2去给keystone发请求，查询revocation list，来判断token1是否已经被revoke了。若未revoke，再用cms解析该token1，得到token_data，然后再判断该token1是否过期。

**pki判断token是否revoke时是否一定需要访问keystone？**
使用pki时keystonemiddleware会先访问keystone获取一个revocation list，然后保存在本地，并设置一个revocation_cache_time。若过了该时间，下次验证时会重新去keystone获取。

若token1为uuid、fernet，会先去keystone获取一个新的token2，然后用token2去给keystone发请求，验证token1是否有效。

# keystone验证token #

----------
对于uuid会先从数据库中查找token，再判断token是否已过期，再判断token是否被revoke了。
对于fernet会先用key来解码token，再判断token是否已过期，再判断token是否被revoke了。















