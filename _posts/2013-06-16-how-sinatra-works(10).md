---
layout: post
title: "How sinatra works(10)"
description: ""
category:
tags: [ruby, sinatra]
---

前面简单介绍了一些 Sinatra 的 filter 的原理，今天介绍一下 route 的实现方式，还是前面的 hello, world 程序。

<!--break-->

{% highlight ruby %}
# app.rb

require 'sinatra'

get '/' do
  'hello, world'
end
{% endhighlight %}

首先看看 get 方法:

{% highlight ruby %}
# sinatra/base.rb line 1365

# Defining a `GET` handler also automatically defines
# a `HEAD` handler.
def get(path, opts = {}, &block)
  conditions = @conditions.dup
  route('GET', path, opts, &block)

  @conditions = conditions
  route('HEAD', path, opts, &block)
end
{% endhighlight %}

根据注释，当我们定义一个 GET 路由的时候，会自动定义一个 HEAD 的路由。下面是 route 方法:

{% highlight ruby %}
# sinatra/base.rb line 1385

def route(verb, path, options = {}, &block)
  # Because of self.options.host
  host_name(options.delete(:host)) if options.key?(:host)
  enable :empty_path_info if path == "" and empty_path_info.nil?
  signature = compile!(verb, path, block, options)
  (@routes[verb] ||= []) << signature
  invoke_hook(:route_added, verb, path, block)
  signature
end
{% endhighlight %}

host_name 方法前面有看过; 如果有设置空字符串的路由，比如 `get ' '` 这样的, 那么设置 empty_path_info 为 true。

下面 compile! 方法，讲 filter 的时候也仔细的分析过，和 filter 类似，结果放在 @routes 这个类实例变量里面。最后是 invoke_hook 方法:

{% highlight ruby %}
# sinatra/base.rb line 1395

def invoke_hook(name, *args)
  extensions.each { |e| e.send(name, *args) if e.respond_to?(name) }
end
{% endhighlight %}

调用 sinatra app 拓展的 route_added 回调。

关于 sinatra 的拓展，之后会详细的介绍一下。
