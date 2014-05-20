---
layout: post
title: "How sinatra works(4)"
description: ""
category:
tags: [ruby, sinatra]
---

[上篇博客](/2013/05/26/how-sinatra-works(3\)) 有提到 sinatra 除了 classic style, 还有 modular style, 那么这两种 style 究竟有什么区别，如何实现，并且各自有什么应用场景？

<!--break-->

前面的 hello, world! 的例子就是一个典型的 classic style, 实际上 sinatra 所有的方法都在 sinatra/base 里面，每一个 sinatra 应用都是 Sinatra::Base 的子类，就好象每一个 Rails 应用都是 Rails::Application 的子类一样。

当我们使用 classic style 时，application 就是 Sinatra::Application，在顶层使用的方法都被作用到 Sinatra::Application 中([第一篇文章](/2013/05/22/how-sinatra-works(1\)/)讲解过这个问题), 这样的 DSL 方式让人根本注意不到它的存在, 深藏功与名。除了直接通过 ruby app.rb 这样简单暴力的方式启动 sinatra 的方式之外，还可以使用 rackup 通过 rack builder 来启动它，方法大致如下:

{% highlight ruby %}
# app.rb

require 'sinatra'

get '/' do
  'hello, world!'
end

# config.ru

require File.expand_path('../app', __FILE__)

run Siantra::Application
{% endhighlight %}

当执行 rackup 时，app_file != $0，从而使应用入口不是 Application.run! 而是通过 Rack::Builder 去拼装 application, 然后跑起来。关于 Rack::Builder 的原理这边做不做介绍了。

modular style 其实没什么特别的，只是如果你不愿意用默认的 Sinatra::Application 作为 application 而是想 ‘自立门户’ 那么可以考虑使用这个 style。比如我有一个应用叫做 HelloWorld, 我一定要叫做这个名字，那么可以这么做:

{% highlight ruby %}
# hello_world.rb

require 'sinatra/base'

class HelloWorld < Sinatra::Base
  get '/' do
    'hello, world'
  end

  run! if app_file == $0
end
{% endhighlight %}

注意到，这里 require 的是 'sinatra/base' 而不是 'sinatra'，因为我们不再需要 'sinatra/main' 中关于 Sinatra::Application 的内容了, 而是仿照它重新写了一些内容，app_file 是在继承 Sinatra::Base 时 inherited 回调设置的。

你会发现我们重写的这个内容不如默认的 Application 功能齐全，比如一些常用参数的设置都没有。一个解决方法是你可以把这些功能复制过来（如果你不嫌折腾的话），另一个方法就是用 rackup 方式启动，因为 rackup 支持很多的参数。

为什么 sinatra 要提供这两种？

classic style 是被推荐的，足够简单精炼, 但是它有局限性，就是当你使用这种方式时你只能有一个 sinatra application，虽然这已经足够做好多事，但是如果一旦应用规模变大，将很难控制。sinatra 绝不是只能写写小应用的玩意儿。modular style 可以将大的应用拆分成多个 sinatra application, 然后通过 rack builder 来处理，比如下面的例子:

{% highlight ruby %}
# hello.rb

class Hello < Sinatra::Base
  get '/' do
    'hello'
  end
end

# world.rb

class World < Sinatra::Base
  get '/' do
    'world'
  end
end

# config.ru

map '/hello' do
  run Hello
end

map '/world' do
  run World
end
{% endhighlight %}

这样以来，答应得到拆分，每块处理好自己的就 ok，就好比 Rails 的各个 Controller。

Rails 也是一个出色的框架，给了我们很多很棒的最佳实践，但是给的太多也许也不是好事，好多东西永远用不到只会成为累赘，如果使用 sinatra 做一些加法，从 Rails 等好的 “重量级” 框架中借鉴他们的优点，也能达到 “知其所以然的” 的效果。
