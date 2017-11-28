---
layout: post
title: openstack的metadata服务
date: 2016-12-19
category: "openstack"
---

## Metadata的概念

在创建虚拟机的时候，用户往往需要对虚拟机进行一些配置，比如：开启一些服务、安装某些包、添加 SSH 秘钥、配置 hostname 等等。

在 OpenStack 中，这些配置信息被分成两类：metadata 和 user data。

Metadata 主要包括虚拟机自身的一些常用属性，如 hostname、网络配置信息、SSH 登陆秘钥等，主要的形式为键值对。而 user data 主要包括一些命令、脚本等。User data 通过文件传递，并支持多种文件格式，包括 gzip 压缩文件、shell 脚本、cloud-init 配置文件等。

虽然 metadata 和 user data 并不相同，但是 OpenStack 向虚拟机提供这两种信息的机制是一致的，只是虚拟机在获取到信息后，对两者的处理方式不同罢了。所以下文统一用 matadata 来描述。

## OpenStack Metadata Service 简介及发展

在 OpenStack 中，虚机访问 Metadata 服务都是访问 169.254.169.254 这个 IP 地址，但是 OpenStack 不提供这样一个地址的服务，查看 endpoint 也不能找到这样 IP 地址的信息，那虚机是如何获取 Metadata 服务？并且，为什么会是这个 IP 地址？

首先来看为什么会是这个 IP 地址？一方面 169.254.X.X 是保留 IP，可以用来提供服务而不与实际 IP 冲突；另一方面，Metadata 最早是由亚马逊提出，**亚马逊的服务中，Metadata 就是从 169.254.169.254 提供服务**。很多镜像为了支持亚马逊，将内部获取 Metadata 的 IP 地址写死了。而 **OpenStack 为了兼容这些镜像，也采用了这个 IP 地址作为 Metadata 的服务地址**。

那既然 169.254.169.254 是保留 IP，OpenStack 中的虚机是如何访问到 Metadata 服务？具体的实现机制在 OpenStack 内部也有过变化。OpenStack 中，Metadata 数据实际上是由 Nova 管理提供的。

在早期的 OpenStack 版本中，直接通过 iptables 中的 nat 表，将 169.254.169.254 映射到了真实提供 Metadata 服务的 IP 地址上。可以查看 nova-network 里面的代码：

```python
def metadata_forward():
    """Create forwarding rule for metadata."""
    if CONF.metadata_host != '127.0.0.1':
        iptables_manager.ipv4['nat'].add_rule('PREROUTING',
                                          '-s 0.0.0.0/0 -d 169.254.169.254/32 '
                                          '-p tcp -m tcp --dport 80 -j DNAT '
                                          '--to-destination %s:%s' %
                                          (CONF.metadata_host,
                                           CONF.metadata_port))
    else:
        iptables_manager.ipv4['nat'].add_rule('PREROUTING',
                                          '-s 0.0.0.0/0 -d 169.254.169.254/32 '
                                          '-p tcp -m tcp --dport 80 '
                                          '-j REDIRECT --to-ports %s' %
                                           CONF.metadata_port)
    iptables_manager.apply()
```

自从 Folsom 版本，OpenStack Neutron 引入了 Linux network namespaces，**使得 IP 地址可以在不同的** 
**namespace 中重叠**（overlap）。原有的 Metadata 工作机制将不再适用。最简单的例子，两个具有相同 IP 的虚机请求 Metadata 数据，**Nova 在收到请求之后不知道将 Metadata 数据回发给哪个虚机**，因为两个虚机的 IP 是一样的！

## Metadata Service 详解

### metadata相关配置

```python
/etc/nova/nova.conf:
[DEFAULT]
enabled_apis = osapi_compute,metadata              #使能metadata服务
metadata_listen=0.0.0.0                            #metadata服务监听地址和端口
metadata_listen_port=8775
[neutron]
service_metadata_proxy=true                        #使用neutron来配合实现metadata服务
metadata_proxy_shared_secret=SECRETE_KEY

/neutron/metadata_agent.ini:
[DEFAULT]
nova_metadata_ip=192.168.122.157                   #nova metadata服务地址
nova_metadata_port = 8775
metadata_proxy_shared_secret=SECRETE_KEY
```

### neutron-metadata-agent 简介

neutron-metadata-agent 是从 Grizzly 版本开始引入 OpenStack Neutron。neutron-metadata-agent 就是为了解决 OpenStack Metadata service 在 Linux network namespaces 中工作的问题而引入的。在 OpenStack Metadata service 中，neutron-metadata-agent 的作用就是连接 Nova Metadata 服务和虚机的。

与 Nova Metadata 服务相连，这个很容易理解。从前面的配置项也能看出来，在/neutron/metadata_agent.ini 中配置了需要连接 Nova Metadata 的所有信息。而连接虚机，是通过 neutron-ns-metadata-proxy 实现的。

### neutron-ns-metadata-proxy 简介

neutron-ns-metadata-proxy是运行在namespace中，连接neutron-metadata-agent和虚机的。





#### 通过virtual router连接metadata服务

```shell
/neutron/l3_agent.ini:
enable_metadata_proxy = True
metadata_port = 9697
metadata_proxy_socket = /opt/stack/data/neutron/metadata_proxy
```

当虚机网络连接在router上了时，查看虚机的route table：

```shell
$ route -n
Kernel IP routing table
Destination       Gateway     Genmask          Flags    Metric   Ref   Use    Iface
0.0.0.0           10.0.0.1    0.0.0.0          UG       0        0     0      eth0
169.254.169.254   10.0.0.1    255.255.255.255  UGH      0        0     0      eth0
```

也就是说，发向 169.254.169.254 的请求会先发送到 10.0.0.1，该地址是子网网关，连接在router上。接下来看看 virutal router 中如何处理该请求。先看看 virtual router 的 namespace 中的 iptalbes 规则：

```shell
$ sudo ip netns exec qrouter-d2a71722-0e50-4ab3-bd64-0e8740fa8c1e iptables -t nat -L |grep 169.254
REDIRECT   tcp  --  anywhere             169.254.169.254      tcp dpt:http redir ports 9697
```

可以看到所有发向 169.254.169.254 的请求被转发到了 9697 端口。9697端口信息如下：

```shell
$ sudo ip netns exec qrouter-d2a71722-0e50-4ab3-bd64-0e8740fa8c1e netstat -anp|grep 9697
tcp        0      0 0.0.0.0:9697            0.0.0.0:*               LISTEN      8560/python         
$ sudo ip netns exec qrouter-d2a71722-0e50-4ab3-bd64-0e8740fa8c1e ps -ef|grep 8560
stack     8560     1  0 Nov23 ?        00:00:03 /usr/bin/python /usr/bin/neutron-ns-metadata-proxy --pid_file=/opt/stack/data/neutron/external/pids/d2a71722-0e50-4ab3-bd64-0e8740fa8c1e.pid --metadata_proxy_socket=/opt/stack/data/neutron/metadata_proxy --router_id=d2a71722-0e50-4ab3-bd64-0e8740fa8c1e --state_path=/opt/stack/data/neutron --metadata_port=9697 --metadata_proxy_user=1000 --metadata_proxy_group=1000 --debug --verbose
```

可以看到neutron-ns-metadata-proxy在监听9697端口，也就是说，**虚机中发出的 Metadata 数据请求最终到达了 neutron-ns-metadata-proxy**。

从neutron-ns-metadata-proxy 进程信息可以看到有这样一个参数 :

> metadata_proxy_socket=/opt/stack/data/neutron/metadata_proxy

neutron-ns-metadata-proxy是通过该Unix Domain socket将请求发送出去的。可以看看监听这个 socket 的进程信息：

```shell
$ sudo netstat -anp | grep metadata
unix  2      [ ACC ]     STREAM     LISTENING     162416   6112/python          /opt/stack/data/neutron/metadata_proxy
$ ps -ef|grep 6112
stack     6112  5935  2 Nov23 pts/12   08:32:47 /usr/bin/python /usr/bin/neutron-metadata-agent --config-file /etc/neutron/neutron.conf --config-file=/etc/neutron/metadata_agent.ini
```

可以看出，通过 Unix Domain socket，**neutron-ns-metadata-proxy 将虚机的请求发送到了neutron-metadata-agent**。结合前面的描述，虚机的请求最终会发送到 Nova Metadata 服务。

**neutron-ns-metadata-proxy 进程会随 virtual router 的创建而创建，同时也会随 virtual router 的移除而终止。**



#### 通过dhcp连接metadata服务

```
/neutron/dhcp_agent.in:
enable_metadata_network = True
enable_isolated_metadata = True
```

当虚机所在网络没有连接到virtual router时，neutron通过dhcp namespace来连接metadata服务和虚机。

查看虚机的route table：

```shell
$ route -n
Kernel IP routing table
Destination       Gateway     Genmask          Flags    Metric   Ref   Use    Iface
0.0.0.0           10.0.0.1    0.0.0.0          UG       0        0     0      eth0
169.254.169.254   10.0.0.2    255.255.255.255  UGH      0        0     0      eth0
```

可以看出，会直接把 Metadata 数据请求发送到 dhcp server 上。在 dhcp 的 namespace 里面，neutron-ns-metadata-proxy 直接监听在 80 端口，所以在 iptables 里面看不到什么特别的规则。neutron-ns-metadata-proxy 与 neutron-metadata-agent 的连接与前面一样，这里就不再赘述了。

为什么这里 neutron-ns-metadata-proxy 可以直接监听在 80 端口，而前面不行？因为 virtual router 的 namespace 要承接 L3 服务，80 端口作为一个常用的端口如果被 Metadata 占用将非常不方便。而在 dhcp 的 namespace 中，80 端口本来没有使用，就没有必要做转发，可以直接使用。

在 dhcp namespace 中，neutron-ns-metadata-proxy 进程会随 network 的 dhcp server 的创建而创建。同样的，从用户的角度不用考虑管理 neutron-ns-metadata-proxy 进程。



## 总结

因为 Linux network namespaces 的引入，使得原本的简单通过 iptables 转发 169.254.169.254 来连接 Metadata 服务不再适用。Neutron 通过自身的实现，将虚机在 namespace 中请求 Metadata 数据经过一系列的转发，最终与 Nova Metadata 服务连接。

其流程图如下：

![metadata](E:\private\blog\_posts\image\metadata.png)





