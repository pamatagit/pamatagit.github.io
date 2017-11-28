---
layout: post
title: openstack对接vmware
date: 2016-11-29
category: "openstack"
---

VMware vCenter Driver可以使nova-compute连接到vcenter，并通过vcenter管理vmware主机集群。整体架构如下： ![vmware-nova-driver-architecture](image\vmware-nova-driver-architecture.jpg)

上述架构中，一个nova-compute服务对应一个主机集群，因此对于nova-scheduler来说，调度的时候是在集群的层面调度的，集群内部的调度是vcenter自己完成的。

vcenter driver也会与glance服务交互，第一次使用某镜像创建虚拟机的时候会从glance下载该镜像到vcenter的datastore。

## 安装vcenter环境

先通过esxi安装镜像安装一台esxi服务器。为了能通过vnc访问虚拟机，需要将esxi服务器的firewall关闭：

```shell
[root@esxi:~]esxcli network firewall set --enabled false
[root@esxi:~]esxcli firewall get
	Default Action: Drop
	Enabled: false
	Loaded: true
```

下载vcenter安装镜像*VMware-VCSA-all-6.0.0-3634788.iso*，执行*vcsa-setup.html*，该安装程序会在esxi服务器上创建一台虚拟机，该虚拟机会作为vcenter server。

安装好后，需要在vcenter中创建datacenter和cluster，在cluster中添加esxi主机，以及datastore。

## 配置openstack连接vcenter

配置nova.conf：

```shell
[DEFAULT]
compute_driver = vmwareapi.VMwareVCDriver

[vmware]
host_ip = 192.168.1.68                           <vCenter host IP>
host_username = administrator@cloudservice.com   <vCenter username>
host_password = 1qaz@WSX                         <vCenter password>
cluster_name = cluster1                          <vCenter cluster name>
datastore_regex = datacenter1:datastore1         <optional datastore regex>
insecure = False                                 <ssl connection option>
```

## 镜像

vmware需要使用vmdk格式的镜像。可以通过qemu-img convert将qcow2或raw格式的镜像转换成vmdk格式，然后上传到glance中。

```shell
$ qemu-img convert -f raw -O vmdk ubuntu.raw ubuntu.vmdk
$ glance image-create \
  --disk-format vmdk \
  --container-format bare \
  --property vmware_disktype="sparse" \
  --property vmware_adaptertype="ide" \
  --property hypervisor_type="vmware" \
  --property hw_vif_model="E1000" \
  --file ubuntu.vmdk \
  --name ubuntu.vmdk \
  --progress
```

**--property hypervisor_type="vmware"**是告诉openstack，要将该虚拟机创在vmware上。

**--property vmware_disktype="sparse"**和**--property vmware_adaptertype="ide"**是表明镜像的类型。

一般通过qemu-img转换过来的镜像，disktype为sparse，adaptertype为ide。不是通过qemu-img转换过来的镜像，这2个参数会不同。可以通过`head -20 <vmdk file name>`命令来查看相应的信息。

对windows操作系统，需要设置***hw_vif_model="E1000"***，否则会因为网卡驱动的问题进不去操作系统。

## 网络

1.将neutron网络类型配置成vlan。将network_dvs的代码拷贝到控制节点，并在/etc/neutron/plugins/ml2/ml2_conf.ini中添加vmware_dvs这个mechanism_driver。

```python
    [DEFAULT]
    verbose = True
    debug = True

    [ml2]
    type_drivers = flat,vlan,vxlan
    tenant_network_types = vlan
    mechanism_drivers = vmware_dvs,openvswitch
    
    [ml2_type_flat]
    flat_networks = external

    [ml2_type_vlan]
    network_vlan_ranges = provider:2000:2010

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [dvs]
    host_ip = 10.89.153.195
    host_username = administrator@vsphere.local
    host_password = 1qaz@WSX
    dvs_name      = br-int 
    insecure = True
```

2.在vcenter上创建一个名叫br-int的分布式交换机，将该交换机上行链路绑定到一张连通的网卡。

