---
layout: post
title: nova-scheduler
date: 2017-04-20
category: "nova"
---

# nova-scheduler代码结构

```json
├── scheduler/
│   ├── client/
│   ├── filters/
│   ├── weights/
│   ├── __init__.py
│   ├── caching_scheduler.py
│   ├── chance.py
│   ├── driver.py
│   ├── filter_scheduler.py
│   ├── host_manager.py
│   ├── ironic_host_manager.py
│   ├── manager.py
│   ├── rpcapi.py
│   ├── scheduler_options.py
│   └── utils.py
├── filters.py
├── weights.py
```



manager.py中定义nova-scheduler服务的manager，SchedulerManager，处理nova-scheduler收到的消息，主要是select_desination方法。

driver.py中定义SchedulerManager的driver基类，Scheduler。

* caching_scheduler.py中定义CachingScheduler这个driver，继承自Scheduler。

* chance.py中定义ChanceScheduler这个driver，继承自Scheduler。

* filter_scheduler.py中定义FilterScheduler这个driver，继承自Scheduler。主要实现select_desination方法。这个driver也是nova-scheduler的默认driver。

  * host_manager.py中定义Scheduler这个driver的host_manager，HostManager，用来管理宿主机，主要功能是调用filter和weight的handler来实现调度，主要的方法有：
    * get_all_host_states，返回所有能发现的主机信息
    * get_filters_hosts，返回满足过滤条件的主机
    * get_weighted_hosts，对传进去的主机进行权重计算，返回排序好的主机列表
  * ironic_host_manager.py中定义Scheduler这个driver的host_manager，IronicHostManager。

  所以真正底层干活的其实是host_manager。

client/目录中定义了SchedulerQueryClient，供其它模块调用scheduler。

filters/目录中定义了各个filter，当driver使用FilterScheduler时，会使用这些filter来筛选主机。

weights/目录定义了各个weighter，对候选主机从各个维度打分。

filters.py定义了BaseFilterHandler，用来执行各个filter，主要实现get_filtered_objects方法。

weights.py定义了BaseWeightHandler，用来执行各个weight，主要实现get_weighed_objects方法。

# nova-schduler执行流程

1. nova-scheduler从MQ中接收到调度请求，调用`manager.py:SchedulerManager`的`select_destination`
2. SchedulerManager的select_destination调用自己driver的select_destination，默认driver是`filter_scheduler.py:FilterScheduler`来调度
3. FilterScheduler的select_destination方法调用自身类的`_schedule`方法来调度
4. _scheduler方法会调用`host_manager.py:HostManager`的`get_all_host_states`来获取所有可用的主机，然后调用`get_filtered_hosts`筛选主机，`get_weighted_hosts`对宿主机进行权重排序
5. get_filtered_hosts执行filter_handler的get_filtered_objects方法过滤主机，该方法会调用所有的filter_class，执行最终的过滤
6. get_weighted_hosts会执行weight_hanlder的get_weighted_objects方法，计算各主机的权重，然后按权重排序
7. 根据配置的数量（默认为1），选取前面几个主机，然后随机从这几个主机中选择一个作为目标节点返回给compute。