---
title: 使用jenkins在Ubuntu下构建Rails持续集成环境
date: 2016-11-30
---

## 前言
**持续集成(continuous integration)**，就是在敏捷开发中经常提到的CI。  
每一次代码提交更新都要通过CI中的自动化测试，这样可以 **早点发现现有的bug**。其目的在于让 **产品快速迭代的同时，尽可能保持高质量**。  
以我们做Rails的开发为例，为例保证项目的质量，我们都会写一定的自动化测试(例如RSpec、minitest)。在多人协作的项目中，我们会基于当前的开发(develop)分支，开一个新的分支进行开发，在代码合并到develop分支前，跑一遍测试，通过了才能合并。如果没有CI，这样的一个操作都是开发人员手动在本地操作的，这样也会带来一些潜在的问题。首先由于测试都是由手动触发并且在本地运行，所以每次开发人员都必须要记住去跑测试，并且这个操作有时也会是很耗时的。其次每一个人本地的环境不一定是一致的，同样的测试在某一个开发人员本地通过了，在另一个人的PC或者线上并不一定可以通过。在引入了CI之后，每一次提交都会在CI服务器上运行测试，这样不仅将之前需要手动触发的操作自动化，同时也保证了测试是在统一的环境(CI服务器)下运行的。

CI保证了交付的质量和效率，给开发团队提供了极大的帮助。不过现有的CI服务价格通常都十分昂贵(以Travis为例，私有项目的最低价格高达$69/month)。所以很多团队会选择开源的持续集成工具搭建自己的CI服务，其中最出名的就是[Jenkins](https://jenkins.io/)了。社区里Jenkins相关的中文资料不多，我最近正好在做相关的工作，所以就整理成了这篇博客。

## 环境搭建
在这里首先介绍下我使用服务器的一些基本信息，我使用的是digitalocean的$10/month的VPS（最开始时候用的是$5，但是每次使用jenkins进行build时都会因为内存不够而进程崩溃），使用的操作系统是Ubuntu 14.04.5 x64，Jenkins的版本是2.19.4。你也可以选择在本地操作系统直接搭建或者使用Vagrant搭建。
### Jenkins Setup
在Ubuntu上安装jenkins
```sh
$ wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
$ sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
$ sudo apt-get update
$ sudo apt-get install jenkins
```
这几段命令执行完成之后jenkins成功安装在你的机器上并且运行在8080端口上了，同时你系统中也新建了一个名为`jenkins`的用户。
这时你可以在浏览器中访问你服务器的8080端口，页面会提示你系统的初始密码存储在`/var/lib/jenkins/secrets/initialAdminPassword`这个文件里。从里面获取密码粘贴到界面的输入框后就完成了认证。认证之后会让你给你两个安装plugin的选项，我有些选择恐惧症，所以就选择了安装推荐的plugin。
安装完成后会让你填写用户名、密码之类的基本信息，以后从浏览器登录jenkins后台时需要用到。填写完成后就进入了jenkins的dashboard。

由于之后会使用jenkins用户安装Ruby，需要root权限。所以在这里我们给予该用户root权限。
```sh
# 把jenkins用户加入sudo用户组
$ adduser jenkins sudo
# 设置密码（也可以选择在visudo设置NOPASSWD让用户请求sudo权限时不需要输入密码）
$ passwd jenkins
```

### Ruby环境搭建
首先安装Ruby相关的一些依赖
```sh
$ sudo su - jenkins
$ sudo apt-get install autoconf bison build-essential libssl-dev libyaml-dev libreadline6 libreadline6-dev zlib1g zlib1g-dev
```

在这里我使用了`rvm`去管理CI服务器上的Ruby版本。
安装rvm
```sh
$ gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ \curl -sSL https://get.rvm.io | bash
```
将rvm加入shell profile中，在.bashrc文件下加入下面这行
```sh
[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
```

安装项目中的Ruby版本(我的是2.3.1)
```sh
$ rvm install 2.3.1
$ rvm use 2.3.1
# 安装bunlder
$ gem install bundler
```

安装一些gem相关的依赖，因为很多时候我们的rails应用需要JavaScript runtime，所以在这里我也安装了nodejs
```sh
$ sudo apt-get install libcurl3-dev libpq-dev nodejs
```

### 安装PostgreSQL
在我的服务器上我使用PostgreSQL作为数据库
```sh
# 安装postgres
$ sudo apt-get install postgresql postgresql-contrib
# 设置postgres password，这里为了演示我直接设置成'password'
$ sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'password';"
```
接着配置好database.yml文件，在这里我直接在jenkins用户home目录下创建。
```yml
# ~/ci_database.yml

default: &default
  host:    localhost
  adapter: postgresql
  encoding: unicode
  pool: 5

test:
  <<: *default
  database: jenkins_test
  username: postgres
  password: password
```

### ssh配置
由于我们的repo大多数都是用Git进行管理的，所以我们也需要在server上安装git
```sh
$ sudo apt-get install git
```
生成jenkins用户的ssh key
```sh
$ ssh-keygen
```
这里由于我的项目的repo在GitHub上，所以之后我就以集成GitHub为例。在shell中打印出ssh公钥，拷贝到你的Github用户下
```sh
$ cat ~/.ssh/id_ras.pub
```
建立ssh和Github的连接
```sh
$ ssh -T git@github.com
```

## 创建jenkins job
首先进入你的jenkins dashboard，点击页面左上方的`New Item`，在新的页面中输入你的CI job的名称，然后选择`Freestyle project`之后点击ok进入下一步。  
下一个页面中是对你项目的一些配置。选择GitHub project后填写你项目在GitHub中的url。  
接着在`Source Code Management`中选择`Git`，在`Repository URL`填写repo的url，由于我们之前配置好了ssh，所以直接填写ssh url(git@github:username/repo.git)。  
之后选择需要构建的分支，在`Branch Specifier`我填写了空，代表所有分支都要进行构建。  
接着在`Build`中选择`Add build step` -> `Execute shell`，接着在其中添加你的build脚本。  

下面贴出我的build脚本，大家可以针对自己需求自己定制。
```sh
#!/bin/bash -x                                 # 指定执行本段脚本的shell为bash，默认情况会使用sh
source ~/.bashrc                               # 读取rvm
bundle install
cp ~/my_database.yml ./config/database.yml     # 将之前写好的数据库配置yml文件拷贝到当前项目中
RAILS_ENV=test bundle exec rake db:setup       # 初始化数据库
RAILS_ENV=test bundle exec rake test           # 运行测试，我使用的是minitest
```
这时候基本的jenkins job就配置好了，在dashboard中你可以自己点击build手动触发。每一次build都会从Git repo中拉取代码，运行你的build脚本。
不过这对我们来说还是不够的，现在只能通过手动触发build，还并不能在我们每次commit并且push到Git repo后自动完成构建。所以我们需要将我们当前配置好的jenkins task和Git repo进行集成。

## GitHub集成
### Commit后自动构建项目
我们在dashboard中选中我们的Project，点击`Configuration`再次进入配置界面。  
在`Build Triggers`配置触发构建的条件。这里我勾选了`Build when a change is pushed to GitHub`，这样会在每次commit push到GitHub上之后进行构建。这也是我们在项目中的通常做法。不过光在Jenkins中配置了是不够的，我们需要GitHub在每次收到push后通过webhook告知Jenkins可以开始进行构建。  
在GitHub中进入你project的repository，选择`Settings` -> `Webhooks` -> `Add webhook`接着填写你的Jenkins hooks url，这段url就是jenkins server的url加上`/github-webhook`。例：http://my-jenkins-server.com/github-webhook/ 。(Note: 目前我们的Jenkins还运行在8080端口上，不要忘记填上端口号。Jenkins可以通过Nginx配置到80端口上。)
在`Which events would you like to trigger this webhook?` 中选择 `Just the push event`。
配置好后点击`Add webhook`
*Note: 这里需要你的jenkins安装了GitHub plugin，如果你在安装jenkins时选择了安装推荐的plugin，是会给你默认装上的。*

完成后再进行commit和push，你会发现在push之后会自动触发jenkins的build环节。

### 更改GitHub commit状态
使用CI服务集成到Github时，通常有三种状态：
1. pending: 代表构建正在执行中，尚未完成。
2. passed: 代表构建成功。
3. failed: 构建失败，通常可能由某些测试没有通过导致。

如果想显示这些状态，就需要Jenkins将构建的状态进行同步。

#### 生成GitHub access token
这里将Jenkins的build时的一些状态同步到GitHub上，Jenkins向GitHub发送请求去同步这些状态，所以需要GitHub的access_token。
进入你GitHub账号的Settings页面中，选中`Develoer settings` -> `Personal access tokens` -> `Generate access token`。在`Select scopes`中选择token可以操作的scope，选中`repo`和`admin:repo_hook`，之后点击`Generate token`。token生成后复制到你的剪贴板上，我们在Jenkins的配置中会用到它。

#### 配置jenkins GitHub Configuration
进入Jenkins的dashboard，点击左侧的`Manage Jenkins` -> `Configure System`，找到`GitHub`这栏，点击Add GitHub Server，之后在Credentials那栏中点击`Add`。在Kind中选择`Secret text`，将之前复制的access token粘贴到Secret那栏，在ID中可以选择给这条credential命名。最后点击`Add`就完成了，然后在Credentials那栏选中你刚刚添加的credential。点击屏幕下方的`Save`保存配置修改。

#### 设置commit status
回到dashboard，进入job的Configuration界面，在`Build`中选择`Add build step` -> `Set build status to "pending" on GitHub commit`。添加后拖动这个build step到之前添加的`Execute shell`之前。这样才能保证在build开始前就把commit status设置成pending。  
之后在`Post-build Actions`中选择`Add post-build action` -> `Set GitHub commit status`。在`Status result`那栏中选择`One of default messages and statuses`。配置好后点击`Save`保存。这样配置完成之后，你在GitHub中每一次commit的状态都会随着CI的状态而改变。

## 结语
持续集成给现在的开发工作带来了很大的便利，将构建操作自动化，可以显著帮助开发者尽早发现现有系统中的问题。完成稳定而又高效的迭代。Jenkins作为一个开源的持续集成软件，可以让我们可以自定义搭建一个免费的持续服务。  
Jenkins中也提供了各种各样强大的插件（例如RVM插件，可以让RVM安装并且使用当前project中指定的Ruby版本），现在的Jenkins自身也支持持续部署、持续交付的功能。个人认为Jenkins对于开发者还是有相当大的学习使用价值的。我在这里也仅仅是一个Getting Started的介绍，有不正确的地方希望大家指正，也希望有感兴趣的朋友以后能在社区里一起交流相关的问题，大家共同进步。

## 参考资料
[Jenkins CI for Rails 4, RSpec, Cucumber, Selenium](http://www.eq8.eu/blogs/6-jenkins-ci-for-rails-4-rspec-cucumber-selenium)  
[Jenkins CI on Ubuntu](https://gist.github.com/ostinelli/b77c20d91e4e33507813)  
