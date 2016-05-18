---
layout: post
date:   2016-5-18
title:  [转载]HostAggregate 与 AvailabilityZone
category: "nova"
---
转自：[availability-zones-and-host-aggregates](https://blog.russellbryant.net/2013/05/21/availability-zones-and-host-aggregates-in-openstack-compute-nova/ "availability-zones-and-host-aggregates")

Confusion around Host Aggregates and Availabaility Zones in Nova seems to be very common. In this post I’ll attempt to show how each are used. All information in this post is based on the way things work in the Grizzly version of Nova.

First, go ahead and forget everything you know about things called Availability Zones in other systems.  They are not the same thing and trying to map Nova’s concept of Availability Zones to what something else calls Availability Zones will only cause confusion.

The high level view is this: A host aggregate is a grouping of hosts with associated metadata.  A host can be in more than one host aggregate.  The concept of host aggregates is only exposed to cloud administrators.

A host aggregate may be exposed to users in the form of an availability zone. When you create a host aggregate, you have the option of providing an availability zone name. If specified, the host aggregate you have created is now available as an availability zone that can be requested.

Here is a tour of some commands.

Create a host aggregate:

    $ nova aggregate-create test-aggregate1
    +----+-----------------+-------------------+-------+----------+
    | Id | Name| Availability Zone | Hosts | Metadata |
    +----+-----------------+-------------------+-------+----------+
    | 1  | test-aggregate1 | None  |   |  |
    +----+-----------------+-------------------+-------+----------+

Create a host aggregate that is exposed to users as an availability zone. (This is not creating a host aggregate within an availability zone! It is creating a host aggregate that is the availability zone!)

    $ nova aggregate-create test-aggregate2 test-az
    +----+-----------------+-------------------+-------+----------+
    | Id | Name| Availability Zone | Hosts | Metadata |
    +----+-----------------+-------------------+-------+----------+
    | 2  | test-aggregate2 | test-az   |   |  |
    +----+-----------------+-------------------+-------+----------+

Add a host to a host aggregate, test-aggregate2. Since this host aggregate defines the availability zone test-az, adding a host to this aggregate makes it a part of the test-az availability zone.

    nova aggregate-add-host 2 devstack
    Aggregate 2 has been successfully updated.
    +----+-----------------+-------------------+---------------+------------------------------------+
    | Id | Name| Availability Zone | Hosts | Metadata   |
    +----+-----------------+-------------------+---------------+------------------------------------+
    | 2  | test-aggregate2 | test-az   | [u'devstack'] | {u'availability_zone': u'test-az'} |
    +----+-----------------+-------------------+---------------+------------------------------------+

Note that the novaclient output shows the availability zone twice. The data model on the backend only stores the availability zone in the metadata. There is not a separate column for it. The API returns the availability zone separately from the general list of metadata, though, since it’s a special piece of metadata.

Now that the test-az availability zone has been defined and contains one host, a user can boot an instance and request this availability zone.

    $ nova boot --flavor 84 --image 64d985ba-2cfa-434d-b789-06eac141c260 \
    > --availability-zone test-az testinstance
    $ nova show testinstance
    +-------------------------------------+----------------------------------------------------------------+
    | Property| Value  |
    +-------------------------------------+----------------------------------------------------------------+
    | status  | BUILD  |
    | updated | 2013-05-21T19:46:06Z   |
    | OS-EXT-STS:task_state   | spawning   |
    | OS-EXT-SRV-ATTR:host| devstack   |
    | key_name| None   |
    | image   | cirros-0.3.1-x86_64-uec (64d985ba-2cfa-434d-b789-06eac141c260) |
    | private network | 10.0.0.2   |
    | hostId  | f038bdf5ff35e90f0a47e08954938b16f731261da344e87ca7172d3b   |
    | OS-EXT-STS:vm_state | building   |
    | OS-EXT-SRV-ATTR:instance_name   | instance-00000002  |
    | OS-EXT-SRV-ATTR:hypervisor_hostname | devstack   |
    | flavor  | m1.micro (84)  |
    | id  | 107d332a-a351-451e-9cd8-aa251ce56006   |
    | security_groups | [{u'name': u'default'}]|
    | user_id | d0089a5a8f5440b587606bc9c5b2448d   |
    | name| testinstance   |
    | created | 2013-05-21T19:45:48Z   |
    | tenant_id   | 6c9cfd6c838d4c29b58049625efad798   |
    | OS-DCF:diskConfig   | MANUAL |
    | metadata| {} |
    | accessIPv4  ||
    | accessIPv6  ||
    | progress| 0  |
    | OS-EXT-STS:power_state  | 0  |
    | OS-EXT-AZ:availability_zone | test-az|
    | config_drive||
    +-------------------------------------+----------------------------------------------------------------+

All of the examples so far show how host-aggregates provide an API driven mechanism for cloud administrators to define availability zones. The other use case host aggregates serves is a way to tag a group of hosts with a type of capability. When creating custom flavors, you can set a requirement for a capability. When a request is made to boot an instance of that type, it will only consider hosts in host aggregates tagged with this capability in its metadata.

We can add some metadata to the original host aggregate we created that was *not* also an availability zone, test-aggregate1.


    $ nova aggregate-set-metadata 1 coolhardware=true
    Aggregate 1 has been successfully updated.
    +----+-----------------+-------------------+-------+----------------------------+
    | Id | Name| Availability Zone | Hosts | Metadata   |
    +----+-----------------+-------------------+-------+----------------------------+
    | 1  | test-aggregate1 | None  | []| {u'coolhardware': u'true'} |
    +----+-----------------+-------------------+-------+----------------------------+

A flavor can include a set of key/value pairs called extra_specs. Here’s an example of creating a flavor that will only run on hosts in an aggregate with the coolhardware=true metadata.

    $ nova flavor-create --is-public true m1.coolhardware 100 2048 20 2
    +-----+-----------------+-----------+------+-----------+------+-------+-------------+-----------+
    | ID  | Name| Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
    +-----+-----------------+-----------+------+-----------+------+-------+-------------+-----------+
    | 100 | m1.coolhardware | 2048  | 20   | 0 |  | 2 | 1.0 | True  |
    +-----+-----------------+-----------+------+-----------+------+-------+-------------+-----------+
    $ nova flavor-key 100 set coolhardware=true
    $ nova flavor-show 100
    +----------------------------+----------------------------+
    | Property   | Value  |
    +----------------------------+----------------------------+
    | name   | m1.coolhardware|
    | ram| 2048   |
    | OS-FLV-DISABLED:disabled   | False  |
    | vcpus  | 2  |
    | extra_specs| {u'coolhardware': u'true'} |
    | swap   ||
    | os-flavor-access:is_public | True   |
    | rxtx_factor| 1.0|
    | OS-FLV-EXT-DATA:ephemeral  | 0  |
    | disk   | 20 |
    | id | 100|
    +----------------------------+----------------------------+

