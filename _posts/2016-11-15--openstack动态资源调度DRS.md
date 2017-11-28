---
layout: post
title: openstack动态资源调度DRS
date: 2016-11-15
category: "nova"
---

# 动态资源调度DRS

DRS服务监控主机的cpu或memory使用率，若超过设定的阈值，则迁移该主机上的虚拟机到其他空闲的主机上，优化主机的资源利用。

基本方案：

1. 获取各主机的资源占用情况，判断其是否需要优化
2. 获取需优化主机上所有虚机的资源占用值，选择占用最少的虚机进行迁移
3. 选择迁移后资源占用率最低的主机，将虚机迁移到该主机

更多需求：

1.  希望有两种模式，一种在全部主机中调度，一种在aggregate内部调度。选择aggregate模式时，不属于任何aggregate的主机如何处理？
2. 希望只有当主机资源使用率长时间超过阈值时，才触发优化



## 获取指标，选择需优化主机

drs服务会周期性地从数据库读取主机的资源状态，将主机的cpu使用率记录在一个列表中。用该列表中的值计算平均使用率，当平均使用率超过阈值，计数+1；若中途有一次平均值没有超过阈值，则清空该计数。当计数达到一定次数时，启动优化。

```python
    def update_host_state(self, context):
        self.host_state = {}
        context = context.elevated()
        nodes = objects.ComputeNodeList.get_all(context)
        for node in nodes:
            # Pick up all the compute metrics reported by monitors
            hostname = node.hypervisor_hostname
            metrics = json.loads(node.metrics)
            avg_used = 0.0
            for metric in metrics:
                name = metric['name']
                if name == 'cpu.percent':
                    metric['value'] = metric['value'] * 100
                    # Caculate the average cpu usage
                    cpu_list = self.cpu_percent_dict.get(host.name, [])
                    cpu_list.append(metric['value'])
                    if len(cpu_list) > constants.METRICS_LIST_LEN:
                        cpu_list.pop(0)
                    csum = 0.0
                    for m in cpu_list:
                        csum += m
                    avg_cpu = csum / len(cpu_list)
                    metric['value'] = avg_cpu
                    self.cpu_percent_dict[hostname] = cpu_list
                	name = name.replace('.', '_')
                	self.host_state[hostname] = metric['value']
```

```python
    def __get_violated_hosts(self, threshhold, stabilization):
        violated_hosts = []
        for host in self.host_state:
            value = host['cpu_percent']

            if value > threshhold:
                count = self.host_and_count[host]
                if not count:
                    count = 1
                else:
                    count += 1
                self.host_and_count[host] = count

                if count >= stabilization and count % stabilization == 0:
                    violated_hosts.append(host)
            else:
                # reset the host_and_count when host recovered
                if host in self.host_and_count:
                    self.host_and_count.pop(host)

        return violated_hosts
```



## 选择待迁移虚机

获取主机上所有虚机，调用ceilometer api查询虚机的cpu使用率，选择使用率最低的虚机。

```python
def __get_instances_for_compute_node(self, context, host, policy):
    instances = objects.InstanceList.get_by_host(
                        context, host,
                        expected_attrs=['metadata', 'system_metadata'])
	return instances.sort(self.__compare_instance_cpu_load)
```

```python
    def __compare_instance_cpu_load(self, instance1, instance2):
        inst_load_1 = self.__get_instance_cpu_load(instance1)
        inst_load_2 = self.__get_instance_cpu_load(instance2)

        if inst_load_1 < inst_load_2:
            return -1
        elif inst_load_1 > inst_load_2:
            return 1
        else:
            return 0
```

```python
def __get_instance_cpu_load(self, instance):
    api = policy_ceilometer_client.API()
    return api.get_instance_ut(instance, constants.CEILOMETER_CPU_UT_KEY)
```

## 选择目的主机

通过scheduler获取所有可用目的主机，计算虚机迁移后各目的主机的cpu使用率，选择使用率最低的主机。





