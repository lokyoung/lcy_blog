---
title: 使用Upstart + Inspeqtor 管理你的Sidekiq
date: 2016-04-17
---

## 前言
最近在工作中碰到一个问题，服务器上的Sidekiq在几天之前就已经挂掉了，而由于我们在服务器端并没有对Sidekiq进行监控和管理，
导致大量的background job停留在队列中无法执行。这无疑是一件相当痛苦的事，所以我开始找寻办法去解决这一问题。
而实际上，Sidekiq的作者就已经给出了最好的解决方案————用Ubuntu自带的Upstart去保证Sidekiq不会因为Ruby虚拟机崩溃等原因停止工作，
用Inspeqtor结合Upstart去监控你的Sidekiq（Sidekiq进程失败时自动重启或者发送邮件通知）。

## Upstart
### 简介
Upstart是Ubuntu系统自带的基于事件的初始化常驻程序，可以在系统开机和关机时启动和关闭服务，在系统运行时对服务进行监督。
Upstart可以让被监督的进程在非正常关闭时，自动重启该进程，保证其在系统中正常稳定地运行。
可以使用系统自带的工具去解决问题，相比较使用第三方工具而言，无疑是更好的方式。所以我选择了Upstart来管理服务器上的Sidekiq进程。  
PS: Upstart除了可以在Ubuntu系统上使用外，在Debian、Fedora、openSUSE、Chrome OS等发行版上也有相关支持。而在最近两年一些Linux发行版的新版本中（Ubuntu 15.04, CentOS 7等），可以使用更先进的Systemd实现Upstart的功能。

### Upstart监督Sidekiq配置
在服务器的/etc/init目录下新建一个`*.conf`文件，在这里我新建了一个名为`sidekiq.conf`的配置文件。
```sh
# /etc/init/sidekiq.conf - Sidekiq config

description "Sidekiq Background Worker"

# 在系统开机时启动服务，关机时关闭服务
start on runlevel [2345]
stop on runlevel [06]

# 在进程崩溃时自动重启进程
respawn
# 30秒之内尝试3次重启，如果失败，则放弃重启
respawn 3 30

# 正常退出所接收到的信号，0(Linux程序正常退出码)
# TERM是通过sidekiqctl停止sidekiq时发送的信号
# 除了这两种信号之外，其余任何终止sidekiq的方式都会触发respawn
normal exit 0 TERM

# 通过reload指令向正在运行的进程发送USR1信号(对于Sidekiq来说，USR1意味着停止接收新的background job)
reload signal USR1

# 通过upstart启动的sidekiq实例的序号
instance $index

script
# this script runs in /bin/sh by default
# respawn as bash so we can source in rbenv
exec /bin/bash <<'EOT'
  # Pick your poison :) Or none if you're using a system wide installed Ruby.
  # rbenv
  # source /home/apps/.bash_profile
  # OR
  # source /home/apps/.profile
  # OR system:
  # source /etc/profile.d/rbenv.sh
  #
  # rvm
  # source /home/apps/.rvm/scripts/rvm

  # Logs out to /var/log/upstart/sidekiq.log by default

  cd /var/www/app
  # 注：这里必须在bundle exec前加上exec，否则会在另外一个进程(不受Upstart监控)中开启sidekiq
  # sidekiq -i 的参数和之前传入是实例序号index一致
  exec bundle exec sidekiq -i ${index} -e production
EOT
end script
```

只需要上面这个简单的配置文件，就完成了Upstart中sidekiq的配置。接下来，我们需要通过Upstart来启动sidekiq，这样才可以让sidekiq进程在Upstart的监督下运行。
```sh
# 由于在系统中可以开启多个sidekiq进程，所以你可以通过指定sidekiq进程的编号方便你对某一个sidekiq进程进行管理
# 下列启动序号为0的sidekiq实例（在我们我应用中，只需要一个sidekiq进程）
# 启动sidekiq
$ sudo service sidekiq start index=0

# 终止sidekiq
$ sudo service sidekiq stop index=0

# 重启sidekiq
$ sudo service sidekiq restart index=0
```
在通过Upstart的方式启动sidekiq之后，sidekiq进程如果遇到异常情况造成崩溃，之后立刻重生（重新启动一个sideiq进程）。你可以尝试使用kill的方式强制关闭sidekiq，会发现sidekiq不会在你的进程列表中消失（pid会改变)。现在你只能通过以上命令中的stop来关闭sidekiq。

### 部署
sidekiq部署的最佳实践，是尽可能早的让sidekiq处于quiet状态（停止接受新的job，只处理当前在队列中的job），并且尽可能晚的重启sidekiq。
在/etc/init/sidekiq.conf中我们曾指定过reload signal USR1，这表示向当前Upstart进程发送reload命令时，传送一个名为USR1的信号过去。USR1是sidekiq系统内置的一个信号量，当sidekiq接收到这个信号时，便会进入quiet状态。  
在你的部署脚本的早期阶段执行以下命令：
```sh
$ sudo service sidekiq reload index=0
```
重启sidekiq也十分地方便，使用Upstart自带的restart即可，restart命令会停止当前任务，之后再次启动。不过restart命令和单纯地先执行stop然后执行start不一样，restart命令会保留当前任务停止时的各项配置，在重新启动时从硬盘中读取这些信息。这样就不会造成如果在重启时，如果正好有任务还在队列中执行，却因为重启而停止导致没有执行完的情况了。
在你的部署脚本中尽量靠后的位置运行重启命令：
```sh
$ sudo service sidekiq restart index=0
```

PS: 由于sudo需要输入密码，而在自动化部署时无法输入密码。而我们启动Upstart是通过service的方式，所以可以让部署用户在执行service命令时不需要输入密码。
打开sudo配置文件：
```sh
$ sudo visudo
```

在打开的配置文件中加上一行（用你的系统用户名替换current_user）
```
current_user ALL=NOPASSWD:/usr/sbin/service
```
