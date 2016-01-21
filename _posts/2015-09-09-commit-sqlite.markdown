---
title: "在Rails中使用sqlite数据库抛出BusyException: database is locked异常的解决方法"
date: 2015-09-09 19:40:20
---
我在学习《Ruby on Rails Tutorial》这本书时，做到用户提交微博时，程序抛出了数据操作的异常。
由于我在做这个demo时使用的是sqlite3数据库，log中的异常信息是SQLite3::BusyException: database is locked: commit transaction。

观察服务器的日志，可以排除该异常是程序本身问题的因素。那就只可能是我的sqlite3配置的问题了。我在stackoverflow上找到了这个问题的解决方法。
由于没找到中文的回答，所以在这里跟大家分享一下。

首先在命令行中进入rails控制台
```sh
$ rails console
```
在命令行中执行以下代码
```sh
ActiveRecord::Base.connection.execute("BEGIN TRANSACTION; END;")
```
之后再执行抛出异常前的操作，程序不再报错，可以正常运行。  

根据stackoverflow上的回答，该操作清除了控制台所占用的事务操作，解放了数据库。在这里由于我还没有找到相关的文档，
所以暂时也不能给出明确的解释。有兴趣的可以去参考[stackoverflow原帖](http://stackoverflow.com/questions/7154664/ruby-sqlite3busyexception-database-is-locked)。
