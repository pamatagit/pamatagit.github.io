# nova evacuate和nova rebuild

*本文所有分析基于Newton版本。*

先看下这2个命令的简介：

```shell
nova evacuate [--password <password>] [--force] <server> [<host>]
Evacuate server from failed host.
```

evacuate是在主机掉电或nova-compute服务down的情况下，将该主机上的虚机在另外一台主机上启动起来。可以配合主机状态检测机制实现虚拟机HA功能。

```
nova rebuild <server> <image>
Shutdown, re-image, and re-boot a server.
```

rebuild是使用另外一个镜像重新生成该虚拟机。镜像可以是另外的操作系统镜像，也可以是该虚机的快照镜像。更多用途是先通过快照备份虚机，然后通过快照恢复虚机。

# evacuate代码分析

## nova-api流程

### EvacuateController._evacuate

入口函数为`nova/api/openstack/compute/evacuate.py:EvacuateController:_evacuate()`

该方法主要是预处理一下api中接收到的参数。

命令`nova evacuate --force instance1 gx-newton-compute`生成的body为：

> {"evacuate": {"host": "gx-newton-compute", "force":  true}}

之前版本中没有force这个参数，所以force默认为None，可以兼容之前的版本的API。现版本中通过novaclient调用api时，force默认为False。

force为None的效果与为True一样。

* 如果指定了host，且force为true，后面就会跳过scheduler环节。
* 如果指定了host，且force为false，后面会以该host为备选，来通过scheduler所有的filter过滤，判断该host是否满足要求
* 如果未指定host，后面会通过scheduler去选择一个主机。

之前版本中有on_shared_storage参数，现版本中没有了，这里加上也是为了兼容之前的版本。现版本默认为None。

* 后面流程中libvirt会自己判断虚机是否使用的共享存储。

### ComputeAPI.evacuate

判断host为有效参数后，调用`nova/compute/api.py:API:evacuate`方法。

```python
#nova/compute/api.py:API
##判断虚机状态
@check_instance_state(vm_state=[vm_states.ACTIVE, vm_states.STOPPED,
                                vm_states.ERROR])
def evacuate(self, context, instance, host, on_shared_storage,
             admin_password=None, force=None):
    LOG.debug('vm evacuation scheduled', instance=instance)
    ##判断主机服务状态
    inst_host = instance.host
    service = objects.Service.get_by_compute_host(context, inst_host)
    if self.servicegroup_api.service_is_up(service):
        LOG.error(_LE('Instance compute service state on %s '
                      'expected to be down, but it was up.'), inst_host)
        raise exception.ComputeServiceInUse(host=inst_host)

    ##更新虚机状态
    instance.task_state = task_states.REBUILDING
    instance.save(expected_task_state=[None])
    self._record_action_start(context, instance, instance_actions.EVACUATE)

    ##生成Migration对象
    migration = objects.Migration(context,
                                  source_compute=instance.host,
                                  source_node=instance.node,
                                  instance_uuid=instance.uuid,
                                  status='accepted',
                                  migration_type='evacuation')
    if host:
        migration.dest_compute = host
    migration.create()

    compute_utils.notify_about_instance_usage(
        self.notifier, context, instance, "evacuate")

    ##获取request_spec
    try:
        request_spec = objects.RequestSpec.get_by_instance_uuid(
            context, instance.uuid)
    except exception.RequestSpecNotFound:
        # Some old instances can still have no RequestSpec object attached
        # to them, we need to support the old way
        request_spec = None

    # NOTE(sbauza): Force is a boolean by the new related API version
    if force is False and host:
        nodes = objects.ComputeNodeList.get_all_by_host(context, host)
        # NOTE(sbauza): Unset the host to make sure we call the scheduler
        host = None
        # FIXME(sbauza): Since only Ironic driver uses more than one
        # compute per service but doesn't support evacuations,
        # let's provide the first one.
        target = nodes[0]
        if request_spec:
            # TODO(sbauza): Hydrate a fake spec for old instances not yet
            # having a request spec attached to them (particularly true for
            # cells v1). For the moment, let's keep the same behaviour for
            # all the instances but provide the destination only if a spec
            # is found.
            destination = objects.Destination(
                host=target.host,
                node=target.hypervisor_hostname
            )
            request_spec.requested_destination = destination

    return self.compute_task_api.rebuild_instance(context,
                   instance=instance,
                   new_pass=admin_password,
                   injected_files=None,
                   image_ref=None,
                   orig_image_ref=None,
                   orig_sys_metadata=None,
                   bdms=None,
                   recreate=True,
                   on_shared_storage=on_shared_storage,
                   host=host,
                   request_spec=request_spec,
                   )
```

1. 判断虚机是否满足evacuate所需条件：

   * 虚机状态必须为ACTIVE、STOPPED、ERROR三者之一
   * 虚机所在主机服务状态必须为down

2. 更新虚机状态

   将虚机task_state设为REBUILDING，并在数据库中记录evacuate这个action。

3. 创建migration对象

   保存在数据库中，在源主机服务重启后查找和清理虚机资源时会用到该对象

4. 获取虚机request_spec

   request_spec在虚机创建时生成并保存在数据库中，scheduler会用到。

   但是老版本中创建的虚机是没有request_spec的，此时request_spec设为None。

   如果需要通过scheduler寻找host，会用到request_spec。如果request_spec不为none，指定了host，且force为false，将host记录在request_spec的requested_destination中，然后将host置为None，后面就会通过scheduler流程来验证该host是否满足要求了。

5. 给conductor发送rpc请求

   注意这里有一个recreate参数指定为True。

   evacuate和rebuild调用的都是conductor的rebuild_instance接口，只是evacuate这里传的recreate为True，rebuild传的recreate为False。

## nova-conductor流程

nova-conductor收到请求后，会调用`nova/conductor/manager.py:ComputeTaskManager:rebuild_instance`方法。

### ComputeTaskManager.rebuild_instance

```python
def rebuild_instance(self, context, instance, orig_image_ref, image_ref,
                     injected_files, new_pass, orig_sys_metadata,
                     bdms, recreate, on_shared_storage,
                     preserve_ephemeral=False, host=None,
                     request_spec=None):

    with compute_utils.EventReporter(context, 'rebuild_server',
                                      instance.uuid):
        node = limits = None
        if not host:
            if not request_spec:
                # NOTE(sbauza): We were unable to find an original
                # RequestSpec object - probably because the instance is old
                # We need to mock that the old way
                filter_properties = {'ignore_hosts': [instance.host]}
                request_spec = scheduler_utils.build_request_spec(
                        context, image_ref, [instance])
            else:
                # NOTE(sbauza): Augment the RequestSpec object by excluding
                # the source host for avoiding the scheduler to pick it
                request_spec.ignore_hosts = request_spec.ignore_hosts or []
                request_spec.ignore_hosts.append(instance.host)
                # NOTE(sbauza): Force_hosts/nodes needs to be reset
                # if we want to make sure that the next destination
                # is not forced to be the original host
                request_spec.reset_forced_destinations()
                # TODO(sbauza): Provide directly the RequestSpec object
                # when _schedule_instances() and _set_vm_state_and_notify()
                # accept it
                filter_properties = request_spec.\
                    to_legacy_filter_properties_dict()
                request_spec = request_spec.to_legacy_request_spec_dict()
            try:
                hosts = self._schedule_instances(
                        context, request_spec, filter_properties)
                host_dict = hosts.pop(0)
                host, node, limits = (host_dict['host'],
                                      host_dict['nodename'],
                                      host_dict['limits'])
            except exception.NoValidHost as ex:
                with excutils.save_and_reraise_exception():
                    self._set_vm_state_and_notify(context, instance.uuid,
                            'rebuild_server',
                            {'vm_state': instance.vm_state,
                             'task_state': None}, ex, request_spec)
                    LOG.warning(_LW("No valid host found for rebuild"),
                                instance=instance)
            except exception.UnsupportedPolicyException as ex:
                with excutils.save_and_reraise_exception():
                    self._set_vm_state_and_notify(context, instance.uuid,
                            'rebuild_server',
                            {'vm_state': instance.vm_state,
                             'task_state': None}, ex, request_spec)
                    LOG.warning(_LW("Server with unsupported policy "
                                    "cannot be rebuilt"),
                                instance=instance)

        try:
            migration = objects.Migration.get_by_instance_and_status(
                context, instance.uuid, 'accepted')
        except exception.MigrationNotFoundByStatus:
            LOG.debug("No migration record for the rebuild/evacuate "
                      "request.", instance=instance)
            migration = None

        compute_utils.notify_about_instance_usage(
            self.notifier, context, instance, "rebuild.scheduled")

        self.compute_rpcapi.rebuild_instance(context,
                instance=instance,
                new_pass=new_pass,
                injected_files=injected_files,
                image_ref=image_ref,
                orig_image_ref=orig_image_ref,
                orig_sys_metadata=orig_sys_metadata,
                bdms=bdms,
                recreate=recreate,
                on_shared_storage=on_shared_storage,
                preserve_ephemeral=preserve_ephemeral,
                migration=migration,
                host=host, node=node, limits=limits)
```

nova-conductor主要做了3件事：

1. 调用nova-scheduler选取host

   当未指定host或force为false时，参数中的host为None，nova-conductor会调用nova-scheduler选取一个合适的host。在调用nova-scheduler之前需要先处理一下request_spec。

   为了兼容性，对于新版本的instances，需要将request_spec转换成老版本的filter_properties和request_spec，然后传给select_destination方法。

   对于老版本的instances，直接生成老版本的filter_properties和request_spec，然后传给select_destination方法。

2. 获取migration

   之前在api中创建了一个migration，并保存到了数据库中。这里直接从数据库中获取，然后作为参数传给compute的rebuild_instance方法

3. 给compute发送rpc请求

## nova-compute流程

