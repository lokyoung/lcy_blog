---
title: ActionCable小记
date: 2016-02-08
---
## 前言
ActionCable作为Rails5中新加入的最重要的功能之一，在Rail5正式发布前就已经引人瞩目。
ActionCable通过使用WebSocket协议，可以实现从Server端主动向Client端推送消息。
同时也在Client端建立了通向Server端的连接，以保证消息推送的安全性和可靠性。  
前一段时间我曾用ActionCable实现了简单的Realtime Notification推送功能和一个简单的chatroom。
Rails5 beta发布时，DHH也曾分享了一个关于ActionCable的一个demo视频。
好的东西需要去分享，我也向大家介绍下我在使用ActionCable时的一些心得。

## 环境搭建
ActionCable在Rails5.0.0之后的版本中都是自带功能。如果你使用的是Rails5 之前的版本。那么你需要在Gemfile中添加actioncable的相关gem。

```ruby
gem 'actioncable'
```

在安装了相关的gem之后，你需要做的第一件事便是定义ActionCable的连接类(ApplicationCable::Connection)。
通过建立这样一个连接，可以进行可靠的认证并且在服务端获取到客户端的信息。在如下的代码中，通过cookies中存取的user_id查找current_user（当前用户）。如果在cookies中找到了相应信息并且返回了current_user，就成功建立连接并且返回current_user，并且current_user可以在actioncable的所有channel instance中直接使用。如果没有找到current_user，认证便会失败。
```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    protected
      def find_verified_user
        if current_user = User.find_by(id: cookies.signed[:user_id])
          current_user
        else
          reject_unauthorized_connection
        end
      end
  end
end
```

之后你需要定义ActionCable中Channel类的基类，你之后要使用的各种channel都应继承与此类。
```ruby
# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

最后你需要给每一个客户端创建相应的consumer实例，注意，其中的url应是server端的actioncable server url(会在后文中配置)。通过客户端创建consumer，可以向server共享client端的cookies。这也是为什么在建立connection时可以获取到cookies的原因。
```coffee
# app/assets/javascripts/cable.coffee
#= require action_cable

@App = {}
App.cable = ActionCable.createConsumer "ws://localhost:28080"
```

<!-- ## Realtime Notification
  Realtime Notification(实时通知)作为一个在主流Web应用中非常常用的一个功能，通过使用WebSocket协议可以从技术上实现。在这里我就简单介绍下如何使用actioncable实现此功能。当然，除了ActionCable外，也有Faye和message-bus等方式。而ActionCable作为Rails5中推出的一个重要特性，将很多底层的操作进行封装，让开发者可以将关注点集中在业务逻辑的实现上。

  同时，这里需要使用Redis的pub/sub功能结合actioncable中的channel。所以你需要在config中配置好你的redis，这里给出一个示例配置。
  ```ruby
  # config/redis/cable.yml

  development: &development
    :url: redis://localhost:6379
    :host: localhost
    :port: 6379
    :timeout: 1
    :inline: true
  test: *development
  ```

  对于本次notification的实现，我的解决方案是系统中每一个用户中在登录认证之后，会subscribe（订阅）一个属于自己的channel，如果server端需要向客户端推送相应信息，只需要向相应用户的channel进行publish就可以。
  在这里定义每一个用户所需要subscribe的channel。在这里，每一个用户会在登录后订阅一个包含自己用户id的频道，这样可以保证只有当前用户才会接受到此频道的消息。(current_user的值从上一节中connection获取)
  ```ruby
  # app/channnels/notices_channel.rb

  class NoticesChannel < ApplicationCable::Channel
    def subscribed
      stream_from "user:#{current_user.id}"
    end
  end
  ```

  在成功订阅收听相应channnel之后，就涉及到我们应当如何向channel发布信息。ActionCable提供了一个非常便捷的方法，ActionCable.server.broadcast，方法有两个参数，第一个参数是你需要发送到的频道，第二个参数为需要发送的消息内容（多为一个hash）。结合上一段中subscribed的channel，我们在这里可以向相应的用户发布所需要发送的信息。以下的代码可以写在你的应用中的任何地方，只要你觉得你需要在那里调用该方法发送消息。
  ```ruby
  ActionCable.server.broadcast "user:#{user.id}", data
  ```

  在客户端成功登陆建立连接，subscribed channel，server端向客户端channel pub message后，最后的一步就是客户端收到信息后该如何进行处理。这里的notification功能我使用了javascripts来在客户端实现。
  在客户端接受到消息后，会触发以下代码中的received这个callback，received方法接收一个参数，便是上一段代码的broadcast方法中你向客户端broadcast的data。(除此之外还有一个connected callback会在成功建立连接时触发)
  ```coffee
  App.notifications = App.cable.subscriptions.create('NoticesChannel', {
    received: function(data) {
      # do anything you want when you received the message
    },
    connected: function() {
      # connected
    }
  });
  ```

## Running
如果你使用的是Rails5.0.0.beta之后的Rails版本，那么你可以直接启动Rails Server。如果你使用的是之前版本的Rails，除了启动Rails Server外，你还需要启动cable server。
cable server是独立于rails server的，官方文档中给出了一个很简单的cable server的配置。
```ruby
# cable/config.ru
require ::File.expand_path('../../config/environment', __FILE__)
Rails.application.eager_load!

run ActionCable.server
```

之后便可以在bin目录中配置启动脚本文件(由于cable server需要单独开启，这里推荐使用多线程的application server，所以使用puma，puma也是rails官方现在推荐的多线程server)
```bash
#!/bin/bash
bundle exec puma -p 28080 cable/config.ru
```

之后便可以在terminal中启动cable server
```bash
$ ./bin/cable
```
使用ActionCable去实现Realtime的一个服务端到客户端的交互，是一种非常简单高效而又安全稳定的方式。希望我的文章能够对大家有帮助。Happy Hacking！ -->
