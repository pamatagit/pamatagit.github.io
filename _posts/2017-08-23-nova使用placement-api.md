---
layout: post
title: nova使用placement-api
date: 2016-12-09
category: "nova"
---

# placement介绍

nova在Newton版本引入了placement API。它是一个单独的API服务，通过wsgi的形式在apache上启动，监听8778端口。

该服务引入了resource provider的概念，代表资源的提供方。resource provider可能是一个计算节点，提供vcpu和ram资源；可能是一个共享存储池，提供disk资源；也可能是一个IP池，提供IP资源。

每种资源均有一个相应的类表示，比如DISK_GB、MEMORY_MB、VCPU等，也可以由用户自定义资源类。

placement会记录每个resource provider中各资源的库存和使用量，库存由inventories记录，使用量由allocation记录。

# placement数据模型

## resource provider

```
MariaDB [nova_api]> desc resource_providers;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| created_at | datetime     | YES  |     | NULL    |                |
| updated_at | datetime     | YES  |     | NULL    |                |
| id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| uuid       | varchar(36)  | NO   | UNI | NULL    |                |
| name       | varchar(200) | YES  | UNI | NULL    |                |
| generation | int(11)      | YES  |     | NULL    |                |
| can_host   | int(11)      | YES  |     | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
```

* uuid：每个resource provder均有一个uuid
* generation：每次resource provider的资源被分配出去时，该值会+1。在更新数据库时该值作为一个判断条件，可以防止同时有多个资源分配请求时，由于资源使用的先后差异，导致数据不一致，引起误判。
* can_host：表示该provider是否可以作为host来启动虚拟机

## inventories

```
MariaDB [nova_api]> desc inventories;
+----------------------+----------+------+-----+---------+----------------+
| Field                | Type     | Null | Key | Default | Extra          |
+----------------------+----------+------+-----+---------+----------------+
| created_at           | datetime | YES  |     | NULL    |                |
| updated_at           | datetime | YES  |     | NULL    |                |
| id                   | int(11)  | NO   | PRI | NULL    | auto_increment |
| resource_provider_id | int(11)  | NO   | MUL | NULL    |                |
| resource_class_id    | int(11)  | NO   | MUL | NULL    |                |
| total                | int(11)  | NO   |     | NULL    |                |
| reserved             | int(11)  | NO   |     | NULL    |                |
| min_unit             | int(11)  | NO   |     | NULL    |                |
| max_unit             | int(11)  | NO   |     | NULL    |                |
| step_size            | int(11)  | NO   |     | NULL    |                |
| allocation_ratio     | float    | NO   |     | NULL    |                |
+----------------------+----------+------+-----+---------+----------------+
```

* resource_provider_id：表示resource provider的唯一标识
* resource_class_id：表示资源类别，各资源的class id不一样
* total：该类资源总量
* reserved：该类资源预留量，比如host需要预留一定memory供操作系统使用
* min_unit：该资源分配的最小量
* max_unit：该资源分配的最大量
* allocation_ratio：分配比例，比如对于vcpu和ram，均可实现超分配
* step_size：资源增量数，比如对于disk资源，min_unit为1G，step_size为5G，那就只能使用1G、6G、11G，而不能使用3G、4G等

> min_unit、max_unit、allocation_ration关系：
>
> 比如一个计算节点有8个物理cpu，即使allocation_ration设为16，也不能给虚拟机分配128个vcpus，而只能设为8，所以需要设置max_unit为8

可以发现该数据表中每条记录表示一个provider的一种resource，当一个provider有多种resource时，该provider会有多条记录。

## allocations

```
MariaDB [nova_api]> desc allocations;
+----------------------+-------------+------+-----+---------+----------------+
| Field                | Type        | Null | Key | Default | Extra          |
+----------------------+-------------+------+-----+---------+----------------+
| created_at           | datetime    | YES  |     | NULL    |                |
| updated_at           | datetime    | YES  |     | NULL    |                |
| id                   | int(11)     | NO   | PRI | NULL    | auto_increment |
| resource_provider_id | int(11)     | NO   | MUL | NULL    |                |
| consumer_id          | varchar(36) | NO   | MUL | NULL    |                |
| resource_class_id    | int(11)     | NO   | MUL | NULL    |                |
| used                 | int(11)     | NO   |     | NULL    |                |
+----------------------+-------------+------+-----+---------+----------------+
```

* consumer_id：使用该资源的对象的id，比如虚拟机的uuid
* resource_provider_id：表示使用的是该resource provider上的资源
* resource_class_id：资源类别
* used：表示资源的使用量

# 配置nova使用placement

在Newton版本，可以选择使用或不使用placement，而在Ocata版本，必须使用placement，否则nova-compute服务启动会失败。

## 1.部署placement api服务

placement api的代码在nova-api的代码中，后期会独立出来，作为一个单独的项目。nova提供了一个nova-placement-api的WSGI脚本，所以placement-api需要通过Apache启动。

```php+HTML
#/etc/httpd/conf.d/placement-api.conf

Listen 8778

<VirtualHost *:8778>
    WSGIDaemonProcess placement-api processes=2 threads=1 user=stack display-name=%{GROUP}
    WSGIProcessGroup placement-api
    WSGIScriptAlias / /usr/bin/nova-placement-api
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%M"
    </IfVersion>
    ErrorLog /var/log/httpd/placement-api.log
</VirtualHost>

Alias /placement /usr/bin/nova-placement-api
<Location /placement>
    SetHandler wsgi-script
    Options +ExecCGI
    WSGIProcessGroup placement-api
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
</Location>
```

## 2.生成数据表

在Newton中，placement使用nova_api的数据库。执行`nova-manage api_db sync`会创建placement相关数据表。

## 3.创建placement用户和service catalog

在keystone中创建placement用户、服务和endpoint。

## 4.配置nova并重启nova-compute和nova-scheduler

```
#/etc/nova/nova.conf

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
user_domain_name = Default
password = x
username = placement
auth_url = http://192.168.2.11:35357/v3
auth_type = password
```

nova-scheduler和nova-compute都会调用placement api，所以控制节点和计算节点都需要配置该配置项。

# nova调用placement过程

1. nova-compute首次启动时，会调用placement在数据库中创建resource provider记录和inventories记录

2. nova-scheduler在创建虚拟机时，会调用placement首先选出在资源上满足要求的resource provider，然后再使用其他filter过滤

   > 不使用placement时，scheduler会获取所有的compute node，然后通过RamFilter、ComputeFilter、DiskFilter等来过滤主机，效率比较低；而使用placement时，直接从数据库中查找满足Ram、Disk、Cpu要求的主机，所以也不需要使用这3个Filter了。

3. 虚拟机创建后，resource tracker会创建allocation记录，并更新resource provider表


