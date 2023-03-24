---
layout: post
title: 'rsync使用测试'
# subtitle: '通过shell脚本，进行redis批量迁移'
date: 2023-03-24
categories: shell
author: Jason
# cover: 'assets/img/profile.png'
tags: rsync daemon
---

# rsync通过daemon方式运行

作者

**摘要**

试验rsync通过daemon方式运行

目录

## 配置rsync服务

### 配置文件相关参数

#### 全局参数

```shell
在文件中[module]之前的所有参数都是全局参数，当然也可以在全局参数部分定义模块参数，这时候该参数的值就是所有模块的默认值。
port
指定后台程序使用的端口号，默认为873。

motd file
"motd file"参数用来指定一个消息文件，当客户连接服务器时该文件的内容显示给客户，默认是没有motd文件的。

log file
"log file"指定rsync的日志文件，而不将日志发送给syslog。比如可指定为“/var/log/rsyncd.log”。

pid file
指定rsync的pid文件，通常指定为“/var/run/rsyncd.pid”。

syslog facility
指定rsync发送日志消息给syslog时的消息级别，常见的消息级别是：uth, authpriv, cron, daemon, ftp, kern, lpr, mail, news, security, sys-log, user, uucp, local0, local1, local2, local3,local4, local5, local6和local7。默认值是daemon。
```

#### 模块参数

```
主要是定义服务器哪个目录要被同步。其格式必须为“[module]”形式，这个名字就是在rsync客户端看到的名字，其实有点象Samba服务器提供的共享名。而服务器真正同步的数据是通过 path来指定的。我们可以根据自己的需要，来指定多个模块，模块中可以定义以下参数：

comment
给模块指定一个描述，该描述连同模块名在客户连接得到模块列表时显示给客户。默认没有描述定义。

path
指定该模块的供备份的目录树路径，该参数是必须指定的。

use chroot
如果"use chroot"指定为true，那么rsync在传输文件以前首先chroot到path参数所指定的目录下。这样做的原因是实现额外的安全防护，但是缺点是需要以roots权限，并且不能备份指向外部的符号连接所指向的目录文件。默认情况下chroot值为true。

uid
该选项指定当该模块传输文件时守护进程应该具有的uid，配合gid选项使用可以确定哪些可以访问怎么样的文件权限，默认值是"nobody"。

gid
该选项指定当该模块传输文件时守护进程应该具有的gid。默认值为"nobody"。

max connections
指定该模块的最大并发连接数量以保护服务器，超过限制的连接请求将被告知随后再试。默认值是0，也就是没有限制。

list
该选项设定当客户请求可以使用的模块列表时，该模块是否应该被列出。如果设置该选项为false，可以创建隐藏的模块。默认值是true。

read only
该选项设定是否允许客户上载文件。如果为true那么任何上载请求都会失败，如果为false并且服务器目录读写权限允许那么上载是允许的。默认值为true。

exclude
用来指定多个由空格隔开的多个文件或目录(相对路径)，并将其添加到exclude列表中。这等同于在客户端命令中使用--exclude来指定模式，一个模块只能指定一个exclude选项。但是需要注意的一点是该选项有一定的安全性问题，客户很有可能绕过exclude列表，如果希望确保特定的文件不能被访问，那就最好结合uid/gid选项一起使用。

exclude from
指定一个包含exclude模式的定义的文件名，服务器从该文件中读取exclude列表定义。

include
用来指定不排除符合要求的文件或目录。这等同于在客户端命令中使用--include来指定模式，结合include和exclude可以定义复杂的exclude/include规则。

include from
指定一个包含include模式的定义的文件名，服务器从该文件中读取include列表定义。

auth users
该选项指定由空格或逗号分隔的用户名列表，只有这些用户才允许连接该模块。这里的用户和系统用户没有任何关系。如果"auth users"被设置，那么客户端发出对该模块的连接请求以后会被rsync请求challenged进行验证身份这里使用的challenge/response认证协议。用户的名和密码以明文方式存放在"secrets file"选项指定的文件中。默认情况下无需密码就可以连接模块(也就是匿名方式)。

secrets file
该选项指定一个包含定义用户名:密码对的文件。只有在"auth users"被定义时，该文件才有作用。文件每行包含一个username:passwd对。一般来说密码最好不要超过8个字符。没有默认的secures file名，需要限式指定一个(例如：/etc/rsyncd.passwd)。注意：该文件的权限一定要是600，否则客户端将不能连接服务器。

strict modes
该选项指定是否监测密码文件的权限，如果该选项值为true那么密码文件只能被rsync服务器运行身份的用户访问，其他任何用户不可以访问该文件。默认值为true。

hosts allow
该选项指定哪些IP的客户允许连接该模块。客户模式定义可以是以下形式：
单个IP地址，例如：192.167.0.1
整个网段，例如：192.168.0.0/24，也可以是192.168.0.0/255.255.255.0
多个IP或网段需要用空格隔开，“*”则表示所有，默认是允许所有主机连接。

hosts deny
指定不允许连接rsync服务器的机器，可以使用hosts allow的定义方式来进行定义。默认是没有hosts deny定义。

ignore errors
指定rsyncd在判断是否运行传输时的删除操作时忽略server上的IO错误，一般来说rsync在出现IO错误时将将跳过--delete操作，以防止因为暂时的资源不足或其它IO错误导致的严重问题。

ignore nonreadable
指定rysnc服务器完全忽略那些用户没有访问权限的文件。这对于在需要备份的目录中有些文件是不应该被备份者得到的情况是有意义的。

lock file
指定支持max connections参数的锁文件，默认值是/var/run/rsyncd.lock。

transfer logging
使rsync服务器使用ftp格式的文件来记录下载和上载操作在自己单独的日志中。

log format
通过该选项用户在使用transfer logging可以自己定制日志文件的字段。其格式是一个包含格式定义符的字符串，可以使用的格式定义符如下所示：
%h远程主机名
%a远程IP地址
%l文件长度字符数
%p该次rsync会话的进程id
%o操作类型："send"或"recv"
%f文件名
%P模块路径
%m模块名
%t当前时间
%u认证的用户名(匿名时是null)
%b实际传输的字节数
%c当发送文件时，该字段记录该文件的校验码

默认log格式为："%o %h [%a] %m (%u) %f %l"，一般来说,在每行的头上会添加"%t [%p] "。在源代码中同时发布有一个叫rsyncstats的perl脚本程序来统计这种格式的日志文件。

timeout
通过该选项可以覆盖客户指定的IP超时时间。通过该选项可以确保rsync服务器不会永远等待一个崩溃的客户端。超时单位为秒钟，0表示没有超时定义，这也是默认值。对于匿名rsync服务器来说，一个理想的数字是600。

refuse options
通过该选项可以定义一些不允许客户对该模块使用的命令参数列表。这里必须使用命令全名，而不能是简称。但发生拒绝某个命令的情况时服务器将报告错误信息然后退出。如果要防止使用压缩，应该是："dont compress = *"。

dont compress
用来指定那些不进行压缩处理再传输的文件，默认值是*.gz *.tgz *.zip *.z *.rpm *.deb *.iso *.bz2 *.tbz
```



### 配置文件示例

```shell
cat /etc/rsyncd.conf

# 欢迎文件
motd file = /etc/rsyncd.motd

# 用户和组id
#uid = root
#gid = root

# 是否chroot，出于安全考虑建议为yes
use chroot = yes
# 是否记录传输记录
transfer logging = no
# 是否只读，值为true时客户端无法上传
read only = yes
# 是否只写，值为true时客户端无法下载
write only = false
# 默认拒绝所有主机连接
hosts deny = *

# 用户名密码文件，每一行格式是：用户名:密码，例如
# tlanyan:12343112
# 该文件权限必须设置为600，除非strict mode设置为false
#secrets file = /etc/rsyncd.secrets

# 定义名为backup的模块
[backup]
# 模块说明
comment = backup directory
# 模块路径，请求改成自己的
path = /data1/yumdata/
# 允许的主机ip
hosts allow = 192.168.216.0/24
# 允许的用户名
#auth users = root
# 是否允许列出该模块，建议为no
list = no
```



## rsync命令

## 参数解析

```shell
-v, --verbose详细模式输出
-q, --quiet精简输出模式
-c, --checksum打开校验开关，强制对文件传输进行校验
-a, --archive归档模式，表示以递归方式传输文件，并保持所有文件属性，等于-rlptgoD
-r, --recursive对子目录以递归模式处理
-R, --relative使用相对路径信息
-b, --backup创建备份，也就是对于目的已经存在有同样的文件名时，将老的文件重新命名为~filename。可以使用--suffix选项来指定不同的备份文件前缀。
--backup-dir将备份文件(如~filename)存放在在目录下。
-suffix=SUFFIX定义备份文件前缀
-u, --update仅仅进行更新，也就是跳过所有已经存在于DST，并且文件时间晚于要备份的文件。(不覆盖更新的文件)
-l, --links保留软链结
-L, --copy-links想对待常规文件一样处理软链结
--copy-unsafe-links仅仅拷贝指向SRC路径目录树以外的链结
--safe-links忽略指向SRC路径目录树以外的链结
-H, --hard-links保留硬链结
-p, --perms保持文件权限
-o, --owner保持文件属主信息
-g, --group保持文件属组信息
-D, --devices保持设备文件信息
-t, --times保持文件时间信息
-S, --sparse对稀疏文件进行特殊处理以节省DST的空间
-n, --dry-run现实哪些文件将被传输
-W, --whole-file拷贝文件，不进行增量检测
-x, --one-file-system不要跨越文件系统边界
-B, --block-size=SIZE检验算法使用的块尺寸，默认是700字节
-e, --rsh=COMMAND指定使用rsh、ssh方式进行数据同步
--rsync-path=PATH指定远程服务器上的rsync命令所在路径信息
-C, --cvs-exclude使用和CVS一样的方法自动忽略文件，用来排除那些不希望传输的文件
--existing仅仅更新那些已经存在于DST的文件，而不备份那些新创建的文件
--delete删除那些DST中SRC没有的文件
--delete-excluded同样删除接收端那些被该选项指定排除的文件
--delete-after传输结束以后再删除
--ignore-errors及时出现IO错误也进行删除
--max-delete=NUM最多删除NUM个文件
--partial保留那些因故没有完全传输的文件，以是加快随后的再次传输
--force强制删除目录，即使不为空
--numeric-ids不将数字的用户和组ID匹配为用户名和组名
--timeout=TIME IP超时时间，单位为秒
-I, --ignore-times不跳过那些有同样的时间和长度的文件
--size-only当决定是否要备份文件时，仅仅察看文件大小而不考虑文件时间
--modify-window=NUM决定文件是否时间相同时使用的时间戳窗口，默认为0
-T --temp-dir=DIR在DIR中创建临时文件
--compare-dest=DIR同样比较DIR中的文件来决定是否需要备份
-P等同于 --partial
--progress显示备份过程
-z, --compress对备份的文件在传输时进行压缩处理
--exclude=PATTERN指定排除不需要传输的文件模式
--include=PATTERN指定不排除而需要传输的文件模式
--exclude-from=FILE排除FILE中指定模式的文件
--include-from=FILE不排除FILE指定模式匹配的文件
--version打印版本信息
--address绑定到特定的地址
--config=FILE指定其他的配置文件，不使用默认的rsyncd.conf文件
--port=PORT指定其他的rsync服务端口
--blocking-io对远程shell使用阻塞IO
-stats给出某些文件的传输状态
--progress在传输时现实传输过程
--log-format=formAT指定日志文件格式
--password-file=FILE从FILE中得到密码
--bwlimit=KBPS限制I/O带宽，KBytes per second
-h, --help显示帮助信息
```



## 参考网站

[rsync网站](https://linux.die.net/man/1/rsync)

