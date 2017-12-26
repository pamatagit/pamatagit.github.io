---
layout: post
title: nova start和nova stop 流程
date: 2016-10-19
category: "nova"
---



# nova start基本流程

1. 通过cli或界面操作发送start请求
2. nova-api收到请求后去根据instance id去数据库中获取虚机信息
3. 确认用户policy权限
4. 确认instance未被锁定，或用户是admin，否则报InstanceIsLocked异常
5. 确认instance是已创建好的虚机，否则报InstanceNotReady异常
6. 确认instance状态为**stopped**，否则报InstanceInvalidState异常
7. 将instance的任务状态更新为task_states.POWERING_ON，并保存到数据库中
8. 在数据库中为该instance创建一条开机action记录
9. 给nova-compute发送rpc请求
10. nova-compute收到请求后从数据库获取虚机网络信息和块设备信息
11. **nova-compute调用底层驱动对虚机做hard_reboot**
* destroy虚机
* 获取镜像元数据
* 获取磁盘信息
* 利用先前获取的网络信息、块设备信息、元数据、磁盘信息生成新的xml文件
* 重新创建网络设备
* 用新生成的xml文件启动虚机
12. 获取虚机电源状态
13. 更新虚机状态到数据库，vm_state设为active，task_state设为none




# nova stop基本流程

1. 通过cli或界面操作发送stop请求
2. nova-api收到请求后去根据instance id去数据库中获取虚机信息
3. 确认用户policy权限
4. 确认instance未被锁定，或用户是admin，否则报InstanceIsLocked异常
5. 确认instance是已创建好的虚机，否则报InstanceNotReady异常
6. 确认instance状态为**active**或**error**，否则报InstanceInvalidState异常
7. 将instance的任务状态更新为task_states.POWERING_OFF，并保存到数据库中
8. 在数据库中为该instance创建一条关机action记录
9. 给nova-compute发送rpc请求
10. **获取虚拟机电源状态，对虚拟机做graceful shutdown，若重试数次仍未关闭，则使用destroy**
11. 更新虚机状态到数据库，vm_state设为stopped，task_state设为stop



