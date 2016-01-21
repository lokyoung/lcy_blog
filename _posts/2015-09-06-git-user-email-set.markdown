---
title: "关于github上commit后不记录到自己github账号上的问题"
date: 2015-09-06 0:39:42
---

最近发现自己在本地向github上commit并且push后，仓库中记录的提交者会是自己本地用户的名字。而且并不会将本次commit记录在我的github账号中。
但是如果在github上直接更改并commit，就将会记录下来。
我还是一个小白，并不知道为什么。所以查看了github的文档。

文档给出了答案，You haven't added your local Git commit email to your profile。原因其实就是我本地的git配置中的username和email并没有设置成
我的github账号关联的email。所以需要通过git config进行配置。进入terminal，具体配置如下：

```sh
git config --global user.name "username"
git config --global user.email "useremail@domain.com"
```

其中username和useremail@domain.com分别换成你的github用户名和关联邮箱。

这样配置后，你每一次的提交就会记录并关联到你自己的github账号中了。
