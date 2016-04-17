---
title: 使用Upstart + Inspeqtor 管理你的Sidekiq(监控、崩溃自动重启、邮件通知)
date: 2016-04-17
---

## 前言
最近在工作中碰到一个问题，服务器上的Sidekiq挂掉之后，由于我们在服务器端并没有对Sidekiq进行监控和管理，
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

## Inspeqtor

### 简介
Inspeqtor是Sidekiq作者自己写的一个监控工具，可以对系统中的开启的服务（通过init.d, systemd或者upstart启动的进程）进行监控。
同时也可以监控进程的内存占用量和CPU使用量。你可以针对不同的进程配置相应的监控规则，在违反这些规则时重启进程或者向你发送邮件告知。
在之前我们通过配置Upstart启动Sidekiq，所以可以使用Inspeqtor对Sidekiq进程进行监控（这里主要想实现的目标，是在Sidekiq崩溃重启时可以收到邮件通知）。

### 环境搭建和通知邮箱配置
#### 安装
Inspeqtor的安装教程可以在[官方wiki](https://github.com/mperham/inspeqtor/wiki/Installation)中找到，在这里我只列出Ubuntu系统中的安装方式。
```sh
$ curl -L https://bit.ly/InspeqtorDEB | sudo bash
$ sudo apt-get install inspeqtor
```

#### 命令行基本操作
这里同样参考自wiki，有兴趣的朋友可以好好研读。
CentOS 6.5 / Ubuntu with Upstart
```sh
$ initctl start inspeqtor
$ initctl stop inspeqtor
$ initctl restart inspeqtor
```
inspeqtor的log会在/var/log/upstart/inspeqtor.log中输出。

#### 邮箱配置
对发送email的配置在inspeqtor的全局配置文件/etc/inspeqtor/inspeqtor.conf中。
可以通过配置gmail，普通的smtp邮箱和服务器本机的smtp邮件服务三种方式发送邮件。
这里我列出普通smtp邮箱的配置
```
# /etc/inspeqtor/inspeqtor.conf
send alerts via email with
  username bubba,
  password "correct horse battery staple",
  smtp_server smtp.example.com,
  to_email accounting@example.com,
  tls_port 587
```
更多的配置可以查看官方wiki中的[Global Configuration](https://github.com/mperham/inspeqtor/wiki/Global-Configuration)
PS: inspeqtor标准版只能指定一个收件邮箱，如果你想让多个邮箱接收inspeqtor邮件，可以使用邮件组。

### 监控sidekiq
我们之前通过使用Upstart实现了让sidekiq意外崩溃时的自动重生，在这里我们需要的只是在sidekiq崩溃重启时发生邮件通知我们。
由于inspeqtor默认会在其监控下进程的pid改变时发送邮件通知，所以监控sidekiq的配置文件非常简单。
新建配置文件/etc/inspeqtor/services.d/sidekiq.inq。
PS: 按道理说，在这里我们只需要添加一个配置文件即可，不需要添加监控规则（因为pid改变时会自动发送邮件），但是我尝试过，如果不添加一条监控规则，inspeqtor是不会对sidekiq进行监控的。所以我随意添加了一条CPU占用率2次大于95%时发送邮件通知的规则。
```
# /etc/inspeqtor/services.d/sidekiq.inq

check service sidekiq
  if cpu:user > 95% for 2 cycles then alert
```
配置完成后，执行restart指令重新启动inspeqtor便可以在sidekiq进程崩溃时收到邮件。

### inspeqtor部署集成
之前曾经提到过，inspeqtor在进程的pid变化时会触发alert（发送邮件）。而我们在部署时是会对sidekiq进行重启的，那么在默认情况下，部署时会收到inspeqtor发来的通知邮件。很显然，这样的邮件是没有必要的。所以inspeqtor提供了针对部署时的指令，可以在应用部署期间暂停对进程的监控，这样就不会收到不必要的通知了。
```sh
# 在部署开始时运行这条指令（建议在部署脚本的第一步运行）
$ inspeqtorctl start deploy

# 在部署结束时运行(建议在部署脚本的最后执行)
$ inspeqtorctl finish deploy
```
PS: inspeqtorctl默认情况下需要sudo才可以执行，可以通过修改inspeqtor的Upstart配置文件（/etc/init/inspeqtor.conf）指定当前用户所在group，group中的用户执行inspeqtorctl就不需要输入sudo了。
```
# /etc/init/inspeqtor.conf
setgid current_user_group
```
