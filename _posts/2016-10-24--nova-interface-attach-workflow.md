---
layout: post
title: nova interface-attach 流程
date: 2016-10-24
category: "nova"
---

# nova interface-attach流程

1. 通过cli或界面发送请求，请求中可以使用port-id，或使用network-id，或使用network-id及fixed-ip
2. 判断传入的参数是否符合要求，并获取instance信息
3. 确认用户policy权限
4. 确认instance未被锁定，或用户是admin，否则报InstanceIsLocked异常
5. 确认虚机状态为active、stopped、paused其中一种，task state为none
6. 向nova-compute发送rpc请求
7. ​

