---
layout: post
title: "How sinatra works(9)"
description: ""
category:
tags: [ruby, sinatra]
---

上篇博客粗略介绍了静态文件的处理方式，如请求不是静态文件，那么会执行下面的 Sinatra 的 filter。同 Rails 类似，filter 分为 before filter 和 after filter，但是没有 arround filter（我还没在 Rails 中使用过这个方法）。

<!--break-->

添加方法很简单，就是在 Sinatra 的 class scope 使用 before 或 after 方法，后面跟上一段代码快就好了, 以 before 为例:

{% highlight ruby %}
# app.rb

require 'sinatra'

before '/admin/*', host_name: 'example.com' do
  authenticate!
end
{% endhighlight %}

在每个路由为 admin/*，并且 host_name 为 example.com 的请求都会执行这个 filter。下面看看它怎么实现的:

{% highlight ruby %}
# sinatra/base line 1279

# Define a before filter; runs before all requests within the same
# context as route handlers and may access/modify the request and
# response.
def before(path = nil, options = {}, &block)
  add_filter(:before, path, options, &block)
end

# add a filter
def add_filter(type, path = nil, options = {}, &block)
  path, options = //, path if path.respond_to?(:each_pair)
  filters[type] << compile!(type, path || //, block, options)
end
{% endhighlight %}

同 middleware 一样 filters 是类的实例变量，是一个 hash，默认值为 {:before => [], :after => []}, 每次调用都会往里面塞一个 compile! 过的 filter:

{% highlight ruby %}
# sinatra/base line 1406

def compile!(verb, path, block, options = {})
  options.each_pair { |option, args| send(option, *args) }
  method_name             = "#{verb} #{path}"
  unbound_method          = generate_method(method_name, &block)
  pattern, keys           = compile path
  conditions, @conditions = @conditions, []

  [ pattern, keys, conditions, block.arity != 0 ?
      proc { |a,p| unbound_method.bind(a).call(*p) } :
      proc { |a,p| unbound_method.bind(a).call } ]
end
{% endhighlight %}

首先把 option 分解开来，调用 host_name('example.com') 方法:

{% highlight ruby %}
# sinatra/base line 1328

# Condition for matching host name. Parameter might be String or Regexp.
def host_name(pattern)
  condition { pattern === request.host }
end

# sinatra/base line 1301

def condition(name = "#{caller.first[/`.*'/]} condition", &block)
  @conditions << generate_method(name, &block)
end

# sinatra/base line 1399

def generate_method(method_name, &block)
  define_method(method_name, &block)
  method = instance_method method_name
  remove_method method_name
  method
end
{% endhighlight %}

上述代码涉及到了另一个类实例变量 @conditions, 这个变量是一个数组，里面存了一些 unbound method, 方法的内容是 pattern === request.host. 然后再创建一个 unbound_method，方法名为 "before /admin/*"，内容为 before 方法传进来的 block。compile 方法通过一系列巧妙的算法将传入的 path 变成一个正则和一组 keys，正则用于匹配 request 的 path，keys 用于存放通过 request path 传入的参数。

上面例子调用 compile 之后返回值为 [/\A\/admin\/(.*?)\z/, ["splat"]]。

最后 compile! 函数返回一个数组，包括 pattern 也就是 compile 得到的正则，keys 也就是 compile 得到的第二个元素，以及 conditions 和一个 proc，这个 proc 就是你传入的代码块。

通过上述的分析，我们大概可以知道，当一个 request 进来以后，会判断 path 是否符合正则，然后 conditions 是否全部满足，若上面都是肯定的答案，则执行代码块。这是我的猜测，但是否真相就是如此，之后再分析。
