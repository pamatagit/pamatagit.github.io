---
layout: post
title: nova-compute一直报NeutronClientException:Unauthorized错误
date: 2016-08-26
category: "nova"
---

## 问题描述 ##

创建虚拟机失败，查看nova-compute日志发现，失败原因是neutron在创建port的时候出错，错误原因是NeutronClientException：Authentication Failed，重启compute服务后错误消失。

## 问题分析 ##

出现认证失败，说明nova调用neutron api时传给neutronclient的token无效。而重启compute服务后错误消失，可以推断是之前的token过期了导致的。那这个token是哪里来的呢？

查看nova/network/neutronv2/api.py文件，发现有一个全局变量_ADMIN_AUTH，该变量是一个auth_plugin。在nova-compute服务第一次启动时，会读取nova.conf中的[neutron]组下的配置项，生成该plugin。
{% highlight python %}

    global _ADMIN_AUTH
    global _SESSION

    auth_plugin = None

    if not _SESSION:
        _SESSION = session.Session.load_from_conf_options(CONF, NEUTRON_GROUP)

    if admin or (context.is_admin and not context.auth_token):
        if not _ADMIN_AUTH:
            _ADMIN_AUTH = _load_auth_plugin(CONF)

        with lockutils.lock('neutron_admin_auth_token_lock'):
            auth_token = _ADMIN_AUTH.get_token(_SESSION)

        auth_plugin = token_endpoint.Token(CONF.neutron.url, auth_token)

{% endhiglight %}
该plugin会去keystone请求一个token，并用这个token生成一个neutronclient。

之后需要再次生成neutronclient时该plugin会先判断之前获取的token是否过期，若过期则会去keystone重新申请一个token，若未过期则使用先前的token。

那是怎么判断token是否过期的呢？
{% highlight python %}

    def will_expire_soon(self, stale_duration=None):
        stale_duration = (STALE_TOKEN_DURATION if stale_duration is None
                          else stale_duration)
        norm_expires = timeutils.normalize_time(self.expires)

        soon = (timeutils.utcnow() + datetime.timedelta(
                seconds=stale_duration))
        return norm_expires < soon

{% endhighlight %}
从上段代码可以看出，将本地当前时间与token过期时间对比，若当前时间小于过期时间，则未过期。
这里的token过期时间是keystone在生成token时根据keystone节点的本地时间计算出来的。
如果keystone节点的时间与compute节点的时间不同步，比如compute节点比keystone节点早一天，compute节点的本地时间总是会小于token的过期时间。这样compute就会将已过期的token判断成未过期的，并用这个token来生成neutronclient。
neutronclient调用neutron api时会先去keystone验证其token的有效性，由于这个token在keystone中是已经过期了的，就会报认证失败的错误。

总结：compute节点和keystone节点时间不同步，会导致compute判断token是否过期是出错。

## 解决方法 ##

同步keystone和compute节点的时间，最好使用ntp服务。

  