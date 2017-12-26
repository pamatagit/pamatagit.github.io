---
layout: post
title: nova服务状态更新
date: 2016-08-26
category: "nova"
---

*声明：本文的所有分析都是基于kilo版本，且由于工作中没有用到zookeeper，所以本文不涉及zookeeper。*

通过nova service-list可以查看到nova各个服务的状态，那各个服务状态是up还是down是怎么决定的呢？

跟踪nova service-list的api，该api最终会调用nova/api/openstack/compute/contrib/services.py中ServiceController类的_get_services_list方法获取所有的service列表，然后调用_get_service_detail来判断每个service的状态。
{% highlight python %}

    def _get_service_detail(self, svc, detailed):
        alive = self.servicegroup_api.service_is_up(svc)
        state = (alive and "up") or "down"
        active = 'enabled'
        if svc['disabled']:
            active = 'disabled'
        service_detail = {'binary': svc['binary'], 'host': svc['host'],
                     'zone': svc['availability_zone'],
                     'status': active, 'state': state,
                     'updated_at': svc['updated_at']}
        if self.ext_mgr.is_loaded('os-extended-services-delete'):
            service_detail['id'] = svc['id']
        if detailed:
            service_detail['disabled_reason'] = svc['disabled_reason']

        return service_detail

    def _get_services_list(self, req, detailed):
        services = self._get_services(req)
        svcs = []
        for svc in services:
            svcs.append(self._get_service_detail(svc, detailed))

        return svcs

{% endhighlight %}
在_get_service_detail中会调用servicegroup_api的service_is_up方法来判断服务的状态。
{% highlight python %}

    def service_is_up(self, member):
        """Check if the given member is up."""
        # NOTE(johngarbutt) no logging in this method,
        # so this doesn't slow down the scheduler
        return self._driver.is_up(member)

{% endhighlight %}
这里的driver有3种:
{% highlight python %}

	'db': 'nova.servicegroup.drivers.db.DbDriver',
    'zk': 'nova.servicegroup.drivers.zk.ZooKeeperDriver',
    'mc': 'nova.servicegroup.drivers.mc.MemcachedDriver'

{% endhighlight %}
默认使用'db'这个driver。使用该driver时，该服务会运行一个周期任务，定时更新数据库中service表的update_at字段。查询服务状态时is_up方法会去比较update_at字段的时间值，与当前时间做比较，看时间间隔是否小于一个设定值，若小于则表明该服务状态正常，就会判断该服务状态为up，反之则为down。

如果要使用'mc'这个driver，则需要在配置文件中添加配置项：

    servicegroup_driver = mc
    memcached_servers = 192.168.1.11,192.168.1.12

且需要安装memcached服务:  `apt-get install memcached python-memcached`

使用这个driver时，该服务会在memcached中保存一个key为host:topic的键值对，并周期性的将当前时间赋给该key。查询服务状态时is_up方法会去memcached中取这个key的值，并与当前时间做比较，看时间间隔是否小于一个设定值，若小于则表明该服务状态正常，就会判断该服务状态为up，反之则为down。

那这个周期任务又是在哪里定义的呢？

在服务最初启动时，会执行 `self.servicegroup_api.join(self.host, self.topic, self)`，这里的join最终会调用driver的join方法。

先看dbdriver的join方法：

{% highlight python %}

    def join(self, member, group, service=None):
        """Add a new member to a service group.

        :param member: the joined member ID/name
        :param group: the group ID/name, of the joined member
        :param service: a `nova.service.Service` object
        """
        LOG.debug('DB_Driver: join new ServiceGroup member %(member)s to '
                  'the %(group)s group, service = %(service)s',
                  {'member': member, 'group': group,
                   'service': service})
        if service is None:
            raise RuntimeError(_('service is a mandatory argument for DB based'
                                 ' ServiceGroup driver'))
        report_interval = service.report_interval
        if report_interval:
            service.tg.add_timer(report_interval, self._report_state,
                                 api.INITIAL_REPORTING_DELAY, service)

    def _report_state(self, service):
        """Update the state of this service in the datastore."""
        ctxt = context.get_admin_context()
        state_catalog = {}
        try:
            report_count = service.service_ref['report_count'] + 1
            state_catalog['report_count'] = report_count

            service.service_ref = self.conductor_api.service_update(ctxt,
                    service.service_ref, state_catalog)

            # TODO(termie): make this pattern be more elegant.
            if getattr(service, 'model_disconnected', False):
                service.model_disconnected = False
                LOG.error(_LE('Recovered model server connection!'))

        # TODO(vish): this should probably only catch connection errors
        except Exception:
            if not getattr(service, 'model_disconnected', False):
                service.model_disconnected = True
                LOG.exception(_LE('model server went away'))

{% endhighlight %}

这个join方法通过service.tg.add_timer将_report_state定义为一个定时任务，周期性地执行。该任务会更新数据库中的report_count字段，每次加1.

mcdriver的join方法与dbdriver类似，只是_report_state略有不同。
{% highlight python %}

    def _report_state(self, service):
        """Update the state of this service in the datastore."""
        try:
            key = "%(topic)s:%(host)s" % service.service_ref
            # memcached has data expiration time capability.
            # set(..., time=CONF.service_down_time) uses it and
            # reduces key-deleting code.
            self.mc.set(str(key),
                        timeutils.utcnow(),
                        time=CONF.service_down_time)

            # TODO(termie): make this pattern be more elegant.
            if getattr(service, 'model_disconnected', False):
                service.model_disconnected = False
                LOG.error(_LE('Recovered model server connection!'))

        # TODO(vish): this should probably only catch connection errors
        except Exception:
            if not getattr(service, 'model_disconnected', False):
                service.model_disconnected = True
                LOG.exception(_LE('model server went away'))

{% endhighlight %}
该任务会周期性地更新memcached中service key的值，将其设为当前utc时间。key为topic和host组成，topic为服务名，如nova-scheduler、nova-conductor等，host为该服务所在主机名。

总结：

1.服务在启动时都会定义一个周期性的任务，去上报自身的状态。dbdriver上报到数据库，mcdriver上报到memcached中。

2.查询服务状态时，都会去查询服务最近的更新时间，然后与当前时间做个对比，来判断该服务是up还是down的。只是dbdriver是去数据库中查询最近的更新时间，而mcdriver是去memcached中查询最近的更新时间。




