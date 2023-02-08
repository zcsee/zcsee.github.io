---
layout: post
title: 'CentOS7单用户模式修改root密码'
# subtitle: '通过shell脚本，进行redis批量迁移'
date: 2023-02-06
categories: centos
author: Jason
# cover: 'assets/img/profile.png'
tags: centos7 密码重置
---

# CentOS7单用户模式修改root密码

### 操作概览：

1. 重启后进入grub编辑页面
2. 在linux16这行的行末增加 rw single init=/bin/bash
3. passwd root 命令修改root密码
4. touch /.autorelabel 命令使密码修改生效(selinux为enforcing时适用)
5. exec /sbin/init 重启系统（selinux为disabled时，进入救援模式，需通过systemctl reboot重启系统）、

### 以selinux为enforcing为例，具体带截图的操作步骤如下：

1. 机器重启后，在如下画面按e，进行编辑（这个界面只会停留几秒，如果错过请重启机器）

   ![image-20230208103109282](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208103109282.png)

   > 会进入如下图所示的修改界面

   ![image-20230208103233660](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208103233660.png)

2. 通过“方向键下”来到底，找到倒数第二行（以linux16开头，UTF-8结尾）

   ![image-20230208103443283](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208103443283.png)

   > 在UTF-8后，先敲一个空格，再添加 **rw single  init=/bin/bash**

   ![image-20230208115630741](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208115630741.png)

   > 按**ctrl +x**进行保存后，会来到如下的界面

   ![image-20230208103837157](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208103837157.png)

3. 通过**passwd root **命令，进行root密码的重置，按照提示输入新密码，输入完成后按回车键，需要输入两次

   ![image-20230208104052004](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208104052004.png)

4. 通过**touch /.autorelabel**命令使密码修改生效

   ![image-20230208112540573](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208112540573.png)

5. 通过**exec /sbin/init**命令来重启系统

   ![image-20230208115951240](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208115951240.png)

   ![image-20230208114751888](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208114751888.png)

6. 经过正常的系统重启过程，来到正常登录界面，输入root和新修改的密码，正常登录

   ![image-20230208115106457](C:\Users\szche\AppData\Roaming\Typora\typora-user-images\image-20230208115106457.png)