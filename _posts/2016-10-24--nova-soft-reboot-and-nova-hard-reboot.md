---
layout: post
title: nova reboot和nova reboot --hard 流程
date: 2016-10-24
category: "nova"
---

# nova reboot基本流程

1. 通过cli或界面操作发送reboot请求
2. nova-api收到请求，会先判断reboot type是否为soft或hard
3. 根据instance-id去数据库中获取虚机信息
4. 确认用户policy权限
5. 确认instance未被锁定，或用户是admin，否则报InstanceIsLocked异常
6. 对于soft reboot，确认虚机状态为**active**，且task state为**none**；对于hard reboot，确认虚机状态为active、stopped、paused、suspended、error中的一种，且task state满足要求。
7. 将instance的任务状态更新为rebooting或rebooting_hard，并保存到数据库中
8. 在数据库中为该instance创建一条reboot action记录
9. 给nova-compute发送rpc请求
10. nova-compute收到rpc请求后获取虚机**块设备信息和网络信息**
11. 将instance的任务状态更新为reboot_pending或reboot_pending_hard
12. 将instance的任务状态更新为reboot_started或reboot_started_hard
13. 调用底层reboot接口：
    * 对于soft reboot
      * shutdown虚机
      * start虚机
      * 如果成功，则返回；如果不成功，则尝试调用hard reboot
    * 对于hard reboot
      * destroy虚机
      * 获取镜像元数据
      * 获取磁盘信息
      * 利用先前获取的网络信息、块设备信息、元数据、磁盘信息生成新的xml文件
      * 重新创建网络设备
      * 用新生成的xml文件启动虚机
14. 将instance的任务状态更新为none



# nova reboot实现细节

1. nova-api收到的reuqest请求为：

   ```shell
   req = POST /v2/b76bcea6db2b408a8c10a555fc2610c3/servers/d25af011-fbf7-4b67-a0d4-16166516fe1a/action HTTP/1.0
   Accept: application/json
   Accept-Encoding: gzip, deflate, compress
   Content-Length: 28
   Content-Type: application/json
   Host: 192.168.122.12:8774
   User-Agent: python-novaclient
   X-Auth-Token: ec08a3d5ed0b424a8839488586ef226d
   X-Domain-Id: None
   X-Domain-Name: None
   X-Identity-Status: Confirmed
   X-Project-Domain-Id: default
   X-Project-Domain-Name: Default
   X-Project-Id: b76bcea6db2b408a8c10a555fc2610c3
   X-Project-Name: admin
   X-Role: admin
   X-Roles: admin
   X-Service-Catalog: [{"endpoints": [{"adminURL": "http://192.168.122.12:8774/v2/b76bcea6db2b408a8c10a555fc2610c3", "region": "RegionOne", "internalURL": "http://192.168.122.12:8774/v2/b76bcea6db2b408a8c10a555fc2610c3", "publicURL": "http://192.168.122.12:8774/v2/b76bcea6db2b408a8c10a555fc2610c3"}], "type": "compute", "name": "nova"}...]
   X-Tenant: admin
   X-Tenant-Id: b76bcea6db2b408a8c10a555fc2610c3
   X-Tenant-Name: admin
   X-User: admin
   X-User-Domain-Id: default
   X-User-Domain-Name: Default
   X-User-Id: 4936efa77ed34e45b46c26a68e01fa2c
   X-User-Name: admin

   {"reboot": {"type": "HARD"}}
   ```

2. 权限检查、状态检查都是通过装饰器的方式实现的：

   ```python
   @wrap_check_policy
   @check_instance_lock
   @check_instance_state(vm_state=set(
                   vm_states.ALLOW_SOFT_REBOOT + vm_states.ALLOW_HARD_REBOOT),
                         task_state=[None, task_states.REBOOTING,
                                     task_states.REBOOT_PENDING,
                                     task_states.REBOOT_STARTED,
                                     task_states.REBOOTING_HARD,
                                     task_states.RESUMING,
                                     task_states.UNPAUSING,
                                     task_states.PAUSING,
                                     task_states.SUSPENDING])
   def reboot(self, context, instance, reboot_type):
   	...
   ```

3. 更新数据库中虚机状态：

   ```python
   state = {'SOFT': task_states.REBOOTING,
            'HARD': task_states.REBOOTING_HARD}[reboot_type]
   instance.task_state = state
   instance.save(expected_task_state=expected_task_state)
   ```

   其中`expected_task_state`参数表示数据库中的原始状态必须为expected_task_state中的一种。

4. 保存在数据库中的虚机action记录可以通过命令`nova instance-action-list <server>`和`nova instance-action <server> <request_id>`查看。

5. 块设备信息如下：

   ```python
   {
   	'block_device_mapping': [], 
   	'swap': None, 
   	'ephemerals': [], 
   	'root_device_name': u'/dev/vda'
   }
   ```

   其中block_device_mapping为volume设备映射信息。

6. 网络信息如下：

   ```python
   [VIF({'profile': {}, 
         'ovs_interfaceid': u'1cb1e49d-bf9f-40a5-b139-8ab3a4089940', 
         'preserve_on_delete': False, 
         'network': Network({
             'bridge': 'br-int', 
             'subnets': [Subnet({
                 'ips': [FixedIP({
                     'meta': {}, 
                     'version': 4, 
                     'type': 'fixed', 
                     'floating_ips': [], 
                     'address': u'10.0.0.4'})], 
                 'version': 4, 
                 'meta': {'dhcp_server': u'10.0.0.2'}, 
                 'dns': [], 
                 'routes': [], 
                 'cidr': u'10.0.0.0/24', 
                 'gateway': IP({
                     'meta': {}, 'version': 4, 'type': 'gateway', 
                     'address': u'10.0.0.1'})})], 
             'meta': {'injected': False, 'tenant_id': u'50a487bec3524ca3a7272fda2d2c52a0'},
             'id': u'ee3dc5a4-5e4a-449b-b0ae-29d210db73f5', 'label': u'private'}), 
         'devname': u'tap1cb1e49d-bf', 
         'vnic_type': u'normal', 
         'qbh_params': None, 
         'meta': {}, 
         'details': {u'port_filter': True, u'ovs_hybrid_plug': True}, 
         'address': u'fa:16:3e:34:82:ab', 
         'active': True, 
         'type': u'ovs', 
         'id': u'1cb1e49d-bf9f-40a5-b139-8ab3a4089940', 
         'qbg_params': None})]
   ```

7. soft reboot核心代码：

   ```python
   dom = self._host.get_domain(instance)
   state = self._get_power_state(dom)
   old_domid = dom.ID()
   # NOTE(vish): This check allows us to reboot an instance that
   #             is already shutdown.
   if state == power_state.RUNNING:
       dom.shutdown()
   # NOTE(vish): This actually could take slightly longer than the
   #             FLAG defines depending on how long the get_info
   #             call takes to return.
   self._prepare_pci_devices_for_use(
       pci_manager.get_instance_pci_devs(instance, 'all'))
   for x in xrange(CONF.libvirt.wait_soft_reboot_seconds):
       dom = self._host.get_domain(instance)
       state = self._get_power_state(dom)
       new_domid = dom.ID()

       # NOTE(ivoks): By checking domain IDs, we make sure we are
       #              not recreating domain that's already running.
       if old_domid != new_domid:
           if state in [power_state.SHUTDOWN,
                        power_state.CRASHED]:
               LOG.info(_LI("Instance shutdown successfully."),
                        instance=instance)
               self._create_domain(domain=dom)
   ```

   通过dom.shutdown来关闭虚机，然后通过_create_domain来启动虚机。

8. hard reboot核心代码：

   ```python
   self._destroy(instance)

   image_meta = utils.get_image_from_system_metadata(
       instance.system_metadata)

   if not image_meta and context.auth_token is not None:
       image_ref = instance.get('image_ref')
       image_meta = compute_utils.get_image_metadata(context,
                                                     self._image_api,
                                                     image_ref,
                                                     instance)

   instance_dir = libvirt_utils.get_instance_path(instance)
   fileutils.ensure_tree(instance_dir)

   disk_info = blockinfo.get_disk_info(CONF.libvirt.virt_type,
                                       instance,
                                       image_meta,
                                       block_device_info)
   # NOTE(vish): This could generate the wrong device_format if we are
   #             using the raw backend and the images don't exist yet.
   #             The create_images_and_backing below doesn't properly
   #             regenerate raw backend images, however, so when it
   #             does we need to (re)generate the xml after the images
   #             are in place.
   xml = self._get_guest_xml(context, instance, network_info, disk_info,
                             image_meta,
                             block_device_info=block_device_info,
                             write_to_disk=True)

   # NOTE (rmk): Re-populate any missing backing files.
   disk_info_json = self._get_instance_disk_info(instance.name, xml,
                                                 block_device_info)

   if context.auth_token is not None:
       self._create_images_and_backing(context, instance, instance_dir,
                                       disk_info_json)

   # Initialize all the necessary networking, block devices and
   # start the instance.
   self._create_domain_and_network(context, xml, instance, network_info,
                                   disk_info,
                                   block_device_info=block_device_info,
                                   reboot=True,
                                   vifs_already_plugged=True)
   ```