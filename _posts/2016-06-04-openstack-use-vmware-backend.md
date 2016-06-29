---
layout: post
title: openstack对接vmware实验
date: 2016-06-04
catagory: "nova"
---
## 原理 ##
Openstack支持vmware创建虚拟机，实际是通过vmware提供的vCenter服务实现对vmware的控制，通过vmwareapi.VMwareVCDriver驱动调用vCenter的api实现了vmware的资源信息获取和命令控制功能。具体的实现架构如下图所示：
![](http://i.imgur.com/wSFceka.png)
 
Openstack支持vSphere是以vCenter中的集群（cluster）作为hypervisor的管理单位。所以，在vSphere为backend的环境中，openstack的hypervisor实际上是vCenter中的集群。上图所示，对于openstack的scheduler可以看到3个hypervisor可以供调度。这3个hypervisor实际对应的就是vCenter中的3个集群。

当nova-scheduler调度了某个集群后，其对应的nova-compute服务就会通过vmware driver和vCenter APIS进行互动，将请求落实到vCenter集群中的具体EXSI的node上。在vCenter的集群中，可以通过vCenter的集群DRS，动态资源调度功能，进一步实现调度到EXSI主机上。通过这个过程可以知道，openstack中的vmware处理实际上是对主机资源进行了层次化，启动虚拟机会产生两次调度。一次是openstack层次的，限于集群的调度，另一次则是vsphere内部的调度。这个层次结构类似openstack中的nova-cell设计。

Vmware driver同时会通过glance服务获取镜像，并将镜像拷贝到vsphere的存储池中，这个拷贝过程只会在vsphere本地未找到镜像文件时使用，实际是对image的一种缓存。一旦此image已经缓存了，后续的虚拟机创建就不会再有从openstack拷贝镜像的过程了。

要连通openstack和vmware，首先需要分别安装好openstack环境和vmware环境，vmware使用vcenter管理，并配置好datacenter，cluster和datastore。

## 配置 ##

openstack服务中的glance、nova和cinder均可使用vmware后端。
### glance ###
要在glance中使用vmware，修改`glance-api.conf`，配置好访问vmware的参数：

    [glance_store]
    stores = rbd,http,vmware
    
    default_store = vsphere
    
    vmware_api_insecure = False
    vmware_api_retry_count = 10
    vmware_datastores = Datacenter1:ds1
    vmware_server_host = 10.89.153.195
    vmware_server_username = administrator@vsphere.local
    vmware_server_password = 1qaz@WSX
    vmware_store_image_dir = /openstack_glance

需要事先在vmware的ds1中创建`openstack_glance`目录
### cinder ###
修改`cinder.conf`:

    [DEFAULT]
    enabled_backends = rbd-sas,rbd-ssd,vmware-vmdk
    [vmware-vmdk]
    volume_driver = cinder.volume.drivers.vmware.vmdk.VMwareVcVmdkDriver
    volume_backend_name = vmware_vmdk
    vmware_api_retry_count = 10
    vmware_host_ip = 10.89.153.195
    vmware_host_username = administrator@vsphere.local
    vmware_host_password = 1qaz@WSX
    vmware_volume_folder = cinder-volumes
如果有多个vcenter，可以在enabled_backends中再加一项"vmware-vmdk-1"，然后新增一个section 【vmware-vmdk-1】，并在该section下配置各个参数。
### nova ###
修改nova-compute节点的`nova-compute.conf`:

    [DEFAULT]
    #compute_driver=libvirt.LibvirtDriver
    compute_driver=vmwareapi.VMwareVCDriver
    [libvirt]
	#virt_type=kvm
    virt_type=vmware

再修改`nova.conf`:

    [vmware]
    host_ip = 10.89.153.195
    host_username = administrator@vsphere.local
    host_password = 1qaz@WSX
    cluster_name = cluster1

可以选择多个cluster，用逗号隔开就行。

## 网络 ##
如果openstack使用neutron，且不用其它插件，要使openstack跟vmware网络连通，neutron的物理网络必须使用vlan，然后在vmware中创建一个分布式交换机，在交换机上创建一个名为br-int的端口组，并使能vlan，且指定其vlan号与openstack中私有网络的vlan号相同。

![](http://i.imgur.com/BpxERxc.png)
![](http://i.imgur.com/3wK77Hq.png)




