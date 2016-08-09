---
layout: post
title: nova-compute处理libvirt事件
date: 2016-06-24
category: "nova"
---
## 连接libvirt并注册事件 ##

nova-computer服务启动时会加载libvirt驱动，根据配置文件中的libvirt.driver决定。

    [DEFAULT]
    compute_driver=libvirt.LibvirtDriver
    [libvirt]
    virt_type=kvm

该驱动的构造函数中会初始化Host类，并连接host的libvirt。

    def __init__(self, virtapi, read_only=False):
        self._host = host.Host(self.uri(), read_only,
                               lifecycle_event_handler=self.emit_event,
                               conn_event_handler=self._handle_conn_event)

    def _get_connection(self):
        return self._host.get_connection()

    _conn = property(_get_connection)

在连接libvirt时，还会注册libvirt事件。

    def _get_new_connection(self):
            wrapped_conn.domainEventRegisterAny(
                None,
                libvirt.VIR_DOMAIN_EVENT_ID_LIFECYCLE,
                self._event_lifecycle_callback,
                self)

这里只注册了libvirt.VIR_DOMAIN_EVENT_ID_LIFECYCLE类型的libvirt事件，并定义了该事件的回调方法_event_lifecycle_callback，该回调方法会将事件信息放入一个evnet_queue。

## 开始监听libvirt事件  ##

nova-compute加载完driver之后会调用init_host方法来初始化host，该方法会启动线程监听libvirt事件。

	#compute.manager.py
    def init_host(self):
        """Initialization for a standalone compute service."""
        self.driver.init_host(host=self.host)
        self.init_virt_events()

	#virt.libvirt.driver.py
    def init_host(self, host):
        self._host.initialize()

	#virt.libvirt.host.py
    def initialize(self):
        libvirt.registerErrorHandler(self._libvirt_error_handler, None)
        libvirt.virEventRegisterDefaultImpl()
        self._init_events()

    def _init_events(self):
        self._event_thread = native_threading.Thread(
            target=self._native_thread)
        self._event_thread.setDaemon(True)
        self._event_thread.start()

        eventlet.spawn(self._dispatch_thread)

    def _native_thread(self):
        while True:
            libvirt.virEventRunDefaultImpl()

通过一层层调用可以看到最后会起一个_native_thread线程，循环运行libvirt.virEventRunDefaultImpl方法，该方法就是用来监听libvirt事件的，在此之前必须先执行libvirt.virEventRegisterDefaultImpl方法。
除了监听libvirt事件的线程外，还有一个_dispath_thread，该线程用于读取event_queue中内容。

## 处理事件 ##
当有虚拟机启动、关闭等libvirt事件产生时，_native_thread会监听到该事件并调用该类事件的回调方法，这里是_event_lifecycle_callback方法。该方法会判断该事件类型，若是stop、start、suspend、resume中的一种，则将该事件信息放入event_queue。_dispath_thread就会读到该事件的信息，并处理。

    def _dispatch_events(self):
        while not self._event_queue.empty():
            try:
                event = self._event_queue.get(block=False)
                if isinstance(event, virtevent.LifecycleEvent):
                    self._event_emit_delayed(event)

最后会调用到_lifecycle_event_handler方法。而该方法是在Host初始化时从libvirtdriver中传过来的，具体的是libvirtdriver的emit_event方法。emit_event方法调用的是_compute_event_callback方法。该方法是在nova-compute执行init_host时赋值的。

	#compute.manager.py
    def init_host(self):
        """Initialization for a standalone compute service."""
        self.driver.init_host(host=self.host)
        self.init_virt_events()

    def init_virt_events(self):
        self.driver.register_event_listener(self.handle_events)

	#virt.driver.py
    def register_event_listener(self, callback):
        self._compute_event_callback = callback

所以最终处理事件的方法是compute manager的handle_events方法。
该方法会将libvirt中虚拟机的状态同步到openstack中去。




