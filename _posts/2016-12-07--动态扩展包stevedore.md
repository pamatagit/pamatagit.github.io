---
layout: post
title: 动态扩展包stevedore
date: 2016-12-07
category: "python"
---

在openstack中，有很多插件式模块，需要动态加载。比如nova中compute_driver有libvirt、vmware、xen，compute_monitor有cpu_monitor、memory_monitor等，需要在程序运行时来加载。

openstack使用动态扩展包stevedore来实现该功能。stevedore需要配合使用setuptools的entry points来定义和加载插件。

本文以compute_monitor的使用为例，来展示stevedore包的使用。

## 创建插件

要使用插件，需要创建一个基类，预先定义好所有的接口。实现插件时只需要继承该基类，实现所有的接口即可。

```python
nova/compute/monitors/base.py

@six.add_metaclass(abc.ABCMeta)
class MonitorBase(object):
    def __init__(self, compute_manager):
        self.compute_manager = compute_manager
        self.source = None

    @abc.abstractmethod
    def get_metric_names(self):
        raise NotImplementedError('get_metric_names')

    @abc.abstractmethod
    def get_metrics(self):
        raise NotImplementedError('get_metrics')
```

该基类预先定好了get_metric_names和get_metrics接口，插件只需实现这2个接口就可以了。

cpu monitor的实现如下：

```python
nova/compute/monitors/cpu/virt_driver.py

class Monitor(base.MonitorBase):
    """CPU monitor that uses the virt driver's get_host_cpu_stats() call."""

    def __init__(self, resource_tracker):
        super(Monitor, self).__init__(resource_tracker)
        self.source = CONF.compute_driver
        self.driver = resource_tracker.driver
        self._data = {}
        self._cpu_stats = {}

    def get_metric_names(self):
        return set([
            fields.MonitorMetricType.CPU_FREQUENCY,
            fields.MonitorMetricType.CPU_USER_TIME,
			...
        ])

    def get_metrics(self):
        metrics = []
        self._update_data()
        for name in self.get_metric_names():
            metrics.append((name, self._data[name], self._data["timestamp"]))
        return metrics

    def _update_data(self):
		...
```

ipmi monitor的实现如下：

```python
nova/compute/monitors/ipmi/ipmi_monitors.py

class IPMIMonitor(base.MonitorBase):
    """IPMI monitor."""

    def __init__(self, parent):
        super(IPMIMonitor, self).__init__(parent)
        self.bmc_json = None
        self.source = CONF.compute_driver

    def get_metrics(self):
        metrics = []
        self._update_data()
        for name in self.get_metric_names():
            metrics.append((name, self._data[name], self._data["timestamp"]))
        return metrics

    def get_metric_names(self):
        return set([
            fields.MonitorMetricType.IPMI_BMC,
            fields.MonitorMetricType.IPMI_USER,
            fields.MonitorMetricType.IPMI_PASSWORD,
        ])

    def _update_data(self):
		...
```

## 注册插件

stevedore使用插件时，该插件必须先在entry_points中注册。注册方式是在nova组件的setup.cfg中定义如下配置：

```python
[entry_points]
nova.compute.monitors.cpu =
    virt_driver = nova.compute.monitors.cpu.virt_driver:Monitor
nova.compute.monitors.ipmi =
    ipmi_monitor = nova.compute.monitors.ipmi.ipmi_monitors:IPMIMonitor
```

另一种方式是直接在setup.py中定义：

```
setup(
    entry_points={
        'nova.compute.monitors.cpu': [
            'virt_driver = nova.compute.monitors.cpu.virt_driver:Monitor',
        ],
        'nova.compute.monitors.ipmi': [
          	'ipmi_monitor = nova.compute.monitors.ipmi.ipmi_monitors:IPMIMonitor',
        ]
    },
)
```

`nova.compute.monitors.cpu`是namespace，`virt_driver`是插件名，`nova.compute.monitors.cpu.virt_driver:Monitor`是插件的具体实现。

在安装nova组件后，在`/usr/lib/python2.7/site-packages/nova-**.egg-info/entry_points.txt`文件中会生成如下内容：

```python
[nova.compute.monitors.cpu]
virt_driver = nova.compute.monitors.cpu.virt_driver:Monitor

[nova.compute.monitors.ipmi]
ipmi_monitor = nova.compute.monitors.ipmi.ipmi_monitors:IPMIMonitor
```

## 调用插件

### DriverManager

当有多个driver，只需要使用其中一个时，可以使用这种方式。

```python
mgr = stevedore.driver.DriverManager(
    namespace='nova.compute.monitors.cpu',
    name='virt_driver',
    invoke_on_load=True,
    invoke_args=(resource_tracker,)
)
cpumonitor = mgr.driver
```

`namespace`是插件的命名空间，`name`是插件名，`invoke_on_load`为true表示会实例化该插件类，返回一个类对象，否则返回插件类本身，使用时需要额外实例化，`invoke_args`是插件类实例化时传递给`__init__`的参数。

该方法返回一个`stevedore.driver.DriverManager`对象，该对象的driver属性是该插件类的实例。

```shell
>>> mgr
<stevedore.driver.DriverManager object at 0x1cb9e90>
>>> mgr.driver
<nova.compute.monitors.cpu.virt_driver.Monitor object at 0x24e3290>
```

### ExtensionManager

当有多个extension，都需要加载时可以使用这种方式。

```python
monitors = []
mgr = stevedore.extension.ExtensionManager(
    namespace='nova.compute.monitors.cpu',
    invoke_on_load=True,
    invoke_args=(resource_tracker,)
)
monitors += [ext.obj for ext in plugin_mgr]

def get_data(ext, args):
    print(args)
	print(ext.name, ext.obj.get_metric_names())

mgr.map(get_data,'test for stevedore...')
```

ExtensionManager只需要指定namespace，就会加载该namespace下的所有插件。

调用该插件时使用map(func, *args)方法，对所有插件都执行func方法，将插件类作为参数传给func方法。

### EnabledExtensionManager

当需要筛选出符合条件的extension来加载时，可以使用EnabledExtensionManager。

在nova中用的是这种方式，先通过nova.conf配置需要用的monitor，check_enabeld_monitor读取配置文件，判断需要加载哪些插件。

```python
def check_enabled_monitor(ext):
	return true

monitors = []
mgr = stevedore.enabled.EnabledExtensionManager(
    namespace='nova.compute.monitors.cpu',
    invoke_on_load=True,
    check_func=check_enabeld_monitor,
    invoke_args=(resource_tracker,)
)
monitors += [ext.obj for ext in plugin_mgr]

def get_data(ext, args):
    print(args)
	print(ext.name, ext.obj.get_metric_names())

mgr.map(get_data,'test for stevedore...')
```

需要多传一个check_func参数，用来判断插件是否满足要求，返回true或false。

### NamedExtensionManager

当需要调用指定的插件时，可以使用NamedExtensionManager。

```python
mgr = stevedore.named.NamedExtensionManager(
    namespace='nova.compute.monitors.cpu',
    names=[virt_driver,],
    name_order=true,
    invoke_on_load=True,
    invoke_args=(resource_tracker,)
)
```

需要指定参数names，列出需要加载的插件。如果name_order参数为true，会按照names中的顺序来加载。



整体继承关系：

![stevedore](E:\private\blog\_posts\image\stevedore.png)