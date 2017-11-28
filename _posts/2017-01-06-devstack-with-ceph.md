---
layout: post
title: devstack with ceph
date: 2017-01-06
category: "openstack"
---

在centos7上使用devstack脚本来搭建ceph始终失败，最后选择的方案是先手动搭建ceph cluster，再安装devstack环境来对接ceph。

本文安装ceph是将MON节点和OSD节点装在一台主机上。并且devstack的controller节点也是安装在该主机上。

该主机只配置一张网卡，但是需要配置2块磁盘，一块作为系统盘，一块作为osd磁盘。

# 安装ceph cluster

## 载入ceph安装源

我使用的是公司内部源。

```shell
$ cat /etc/yum.repos.d/extras.repo
[CentOS-extras]
name = CentOS7-extras
baseurl=http://10.121.5.33/rhel7.2-extras
enabled=1
gpgcheck=0

[CentOS-ceph]
name = CentOS7-ceph
baseurl=http://10.121.5.33/ceph
enabled=1
gpgcheck=0
```

## 安装ceph-deploy

> yum install ceph-deploy

## SSH可信授权

> ssh-keygen
> ssh-copy-id -i /root/.ssh/id_rsa.pub node1

其中node1为主机名

## 创建cluster

> mkdir my-cluster
> cd my-cluster
> ceph-deploy new node1  

## 修改配置文件

上述操作会在my-cluster生成一个配置文件：ceph.conf

在配置文件的[global]区增加下面配置:

```shell
osd pool default size = 1
osd pool default min size = 1
public network = 10.9.0.0/24     #hostip所在网络
cluster network = 10.9.0.0/24
```

并且修改：

```shell
mon_host = $hostip
```

将该配置文件拷贝到/etc/ceph/目录下

## 安装ceph

> ceph-deploy install  node1 --no-adjust-repos

## 部署MON

> ceph-deploy mon create-initial

将产生的ceph.client.admin.keyring文件复制到/etc/ceph目录下

## 添加OSD

> ceph-deploy osd create node1:/dev/sdb

检查是否安装成功：

```shell
root@ceph-1:~/my-cluster# ceph -s
    cluster 6aab8be2-95d4-416e-8250-1781547bb598
     health HEALTH_OK
     monmap e1: 1 mons at {node1=10.9.0.21:6789/0}
            election epoch 1, quorum 0 node1
     osdmap e519: 1 osds: 1 up, 1 in
     pgmap v87635: 96 pgs, 1 pools, 2334 MB data, 613 objects
            7310 MB used, 322 GB / 329 GB avail
                 576 active+clean
```

相关的pool和密钥，在安装devstack时，devstack执行脚本会连接ceph来创建

# 安装devstack

## 安装controller节点

```shell
[[local|localrc]]

MULTI_HOST=true
HOST_IP=192.168.104.10 # management & api network
LOGFILE=/opt/stack/logs/stack.sh.log

RECLONE=false

# Credentials
ADMIN_PASSWORD=admin
MYSQL_PASSWORD=x
RABBIT_PASSWORD=x
SERVICE_PASSWORD=x
SERVICE_TOKEN=abcdefghijklmnopqrstuvwxyz

# enable neutron-ml2-vlan
disable_service n-net
enable_service q-svc,q-agt,q-dhcp,q-l3,q-meta,neutron,q-lbaas,q-fwaas,q-vpn

#use linuxbridge instead of openswitch
#if want to use openvwitch, comment this line
Q_AGENT=linuxbridge

#use vlan instead of vxlan 
#if want to use vxlan, comment these lines
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=3001:4000
PHYSICAL_NETWORK=default

#use an existing ceph
enable_plugin devstack-plugin-ceph git://git.openstack.org/openstack/devstack-plugin-ceph
REMOTE_CEPH=True
REMOTE_CEPH_ADMIN_KEY_PATH=/etc/ceph/ceph.client.admin.keyring
```

## 安装compute节点

如果想增加新的compute节点，在controller上先用`virsh secret-list` 命令获取secretid，用`virsh secret-dumpxml secretid`获取secret.xml，用`virsh secret-get-value secretid`获取secretvalue。

在compute节点上使用virsh secret-define --file secret.xml定义secret，然后用`virsh`
`secret-set-value --secret secretid --base64 secretvalue`设置secret值。

这样做是为了保证各节点访问ceph使用同一个secret，不然做迁移时会出错。

并且需要将controller节点的/etc/ceph/ceph.conf和/etc/ceph/ceph.client.admin.keyring文件拷贝到compute节点的/etc/ceph/目录下。

然后使用如下配置安装devstack：

```shell
[[local|localrc]]

MULTI_HOST=true
HOST_IP=192.168.104.11 # management & api network

# Credentials
ADMIN_PASSWORD=admin
MYSQL_PASSWORD=x
RABBIT_PASSWORD=x
SERVICE_PASSWORD=x
SERVICE_TOKEN=abcdefghijklmnopqrstuvwxyz

# Service information
SERVICE_HOST=192.168.104.10
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
Q_HOST=$SERVICE_HOST
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST

DATABASE_TYPE=mysql

ENABLED_SERVICES=n-cpu,q-agt,neutron

Q_AGENT=linuxbridge
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=3001:4000
PHYSICAL_NETWORK=default

# vnc config
NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN
```

安装完之后还要检查/etc/nova/nova.conf中[libvirt]下rbd_secret_uuid这一项是否与设置的一致。

为了加快安装速度，可以配置：

```shell
# use TryStack git mirror
GIT_BASE=http://git.trystack.cn
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
SPICE_REPO=http://git.trystack.cn/git/spice/spice-html5.git
```

## 无密码ssh

另外，为了保证热迁移成功，需要使各节点之间`virsh -c qemu+ssh://stack@gx2/system`命令能够无密码访问。

首先在stack用户下：

> ssh-keygen
>
> ssh-copy-id stack@node1
>
> ssh-copy-id stack@node2

然后切换到root用户下：

> ssh-keygen
>
> ssh-copy-id root@node1
>
> ssh-copy-id root@node2

上面的操作在2个节点都需要执行。