---
layout: post
title: nova-compute中的周期性任务
date: 2017-08-24
category: "nova"
---

## 启动

在nova服务的start方法代码中有以下内容：

```python
if self.periodic_enable:
    if self.periodic_fuzzy_delay:
        initial_delay = random.randint(0, self.periodic_fuzzy_delay)
    else:
        initial_delay = None

    self.tg.add_dynamic_timer(self.periodic_tasks,
                             initial_delay=initial_delay,
                             periodic_interval_max=
                                self.periodic_interval_max)
```

这里表示线程组会加一个定时器，来周期性地执行任务，执行的是self.periodic_tasks方法。

```python
def periodic_tasks(self, raise_on_error=False):
    """Tasks to be run at a periodic interval."""
    ctxt = context.get_admin_context()
    return self.manager.periodic_tasks(ctxt, raise_on_error=raise_on_error)
```

调用的是manager的periodic_tasks，这里的manager是各个服务的manager，都继承自nova.manager.py中的Manager类。

```python
class Manager(base.Base, PeriodicTasks):
    def periodic_tasks(self, context, raise_on_error=False):
        """Tasks to be run at a periodic interval."""
        return self.run_periodic_tasks(context, raise_on_error=raise_on_error)
```

而run_periodic_tasks方法来自父类`PeriodicTasks`，会调用self._periodic_tasks中的任务。

那self._periodic_tasks中的任务是怎么实现的呢？

## 实现

要执行任务，那self._periodic_tasks中应该保存各个任务的方法名，nova是怎么实现的呢？

nova用到了元类和装饰器。

### 装饰器

装饰器负责给需要周期执行的任务方法添加属性，包括`_periodic_task`以及`_periodic_spacing`、`_periodic_enabled`等参数。

```python
def periodic_task(*args, **kwargs):
    def decorator(f):
        # Test for old style invocation
        if 'ticks_between_runs' in kwargs:
            raise InvalidPeriodicTaskArg(arg='ticks_between_runs')

        # Control if run at all
        f._periodic_task = True
        f._periodic_external_ok = kwargs.pop('external_process_ok', False)
        f._periodic_enabled = kwargs.pop('enabled', True)
        f._periodic_name = kwargs.pop('name', f.__name__)

        # Control frequency
        f._periodic_spacing = kwargs.pop('spacing', 0)
        f._periodic_immediate = kwargs.pop('run_immediately', False)
        if f._periodic_immediate:
            f._periodic_last_run = None
        else:
            f._periodic_last_run = now()
        return f
        
    return decorator

#直接使用该装饰器，可给方法增加上述属性  
@periodic_task
def task1(*args, **kwargs):
  pass
```

### 元类

python中一切皆对象，类也是对象，类是元类(metaclass)的实例化对象。

python中内置的元类是type，即python会将class关键字转换成对type的调用。

python中类定义：

```
class MyClass(object):
    pass
```

python会将上述代码翻译成：

```
MyClass = type('Myclass', (), {})
```

type的第一个参数为类名，第二个参数是该类的父类，第三个参数是该类的属性字典。

```
class Foo(object):
	bar = True
	
Foo = type('Foo', (), {'bar': 'True'})
```

方法也可以作为属性传给第三个参数。

`PeriodicTasks`类使用了自定义元类`_PeriodicTasksMeta`，元类筛选`PeriodicTasks`类的属性字典，将属性字典中`_periodic_task`值为true的属性（即类方法）添加到`PeriodicTasks`类的列表属性`_periodic_task`中。

```python
for value in cls.__dict__.values():
    if getattr(value, '_periodic_task', False):
        cls._add_periodic_task(value)
```



创建类时，传给元类的参数中包含那些被装饰器装饰的周期方法，类似：

```python
PeriodicTasks = _PeriodicTasksMeta('PeriodicTasks', (), {'_sync_power_stats': {'_periodic_spacing': 600, '_periodic_enabled': True, '_periodic_name': '_sync_power_states', '_periodic_immediate': True, '_periodic_external_ok': False, '_periodic_task': True, '_periodic_last_run': None}})
```

这些周期方法会被添加到`PeriodicTasks`类的`_periodic_task`中，相当于给该类添加一个`_periodic_task`属性：

```
class PeriodicTasks(object):
    _periodic_task = [('_sync_power_states', <function _sync_power_states at 0x561bed8>)]
```

## 子任务

nova-compute的周期性子任务分别为：

1. _check_instance_build_time

   对build时间超过CONF.instance_build_timeout的虚机，设置为error状态

2. _sync_scheduler_instance_info

   将host上的虚拟机列表同步到scheduler的host_manager中

3. _heal_instance_info_cache

   更新instance的网络信息，每次轮询更新一个

4. _poll_rebooting_instances

   对reboot超时的虚机，调用driver的poll_rebooting_instances，libvirt没有实现

5. _poll_rescued_instances

   配置CONF.rescue_timeout>0时执行。对rescue超时的虚机，执行unrescue操作

6. _poll_unconfirmed_resizes

   配置CONF.resize_confirm_window>0时执行。resize虚拟机后，过了配置时间，从migrations表中找出虚机，执行confirm_resize操作

7. _poll_shelved_instances

   配置CONF.shelved_offload_time>0时执行。对shelved虚机，过了配置时间，执行shelve_offload_instance操作

8. _instance_usage_audit

   虚拟机审计。配置CONF.instance_usage_audit为True，且task_log表中不存在最近一个周期(instance_usage_audit_period，默认是month)的审计记录时时执行。从数据库中获取在一个周期内存在（包括在周期内删除和新创建）的虚拟机信息，并调用notifier.notify()，供ceilometer使用，同时增加task_log表的审计记录。

9. _poll_bandwidth_usage

   调用driver获取所有虚机bandwidth信息，并更新bw_usage_cache表。libvirt未实现，目前只有Xen实现了该方法。

10. _poll_volume_usage

   配置CONF.volue_usage_poll_interval不为0时执行。获取host上所有虚拟机的blockstat信息，并更新到volume_usage_cache表。同时发送到ceilometer。

11. _sync_power_state

    查询driver上虚机的power状态，同步到db中。

12. _reclaim_queued_deletes

    删除过期的soft_deleted虚拟机。

13. update_available_resource

    通过resource_tracker更新主机上的可用资源。

14. _cleanup_running_deleted_instances

    对于db中已经删除，但是在host上任然存在的虚机，执行配置好的操作。包括noop(不操作)、log、shutdown、reap(destroy)

15. _run_image_cache_manager_pass

    把长时间不使用的镜像缓存删除掉

16. _running_pending_deletes

    尝试将db中已经被删除的虚机在host上的文件(instance_path中的文件)清除。若清除失败会重试到一定次数为止。

17. _cleanup_incomplete_migrations

    虚机resize/revert-resize失败时，在源节点或目标节点可能存在残留，该方法将这些残留文件清除。