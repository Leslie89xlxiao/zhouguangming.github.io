---
layout: post
title: "How sinatra works(6)"
description: ""
category:
tags: [ruby, sinatra]
---
Sinatra 是一个 rack 应用，那么他可以随意玩转 rack 的特性。

#### 1. 自定义 ModdleWare

添加 moddleware 的方法非常简单:

<!--break-->

{% highlight ruby %}
# app.rb

require 'sinatra'

use MiddleWare1
use MiddleWare2

get '/' do
  'hello, world!'
end
{% endhighlight %}

原理也很简单：

{% highlight ruby %}
# sinatra/base.rb line 1549

# Use the specified Rack middleware
def use(middleware, *args, &block)
  @prototype = nil
  @middleware << [middleware, args, block]
end
{% endhighlight %}

把 use 的参数加入到类实例变量 @middleware 里面, 然后再实例化的时候把它取出来放在 default moddleware 后面，application 前面。

{% highlight ruby %}
# sinatra/base line 1624

def setup_middleware(builder)
  middleware.each { |c,a,b| builder.use(c, *a, &b) }
end
{% endhighlight %}

#### 2. 把 Sinatra 作为一个 MiddleWare

Sinatra 应用自己也可以作为一个 middleware:

{% highlight ruby %}
# app.rb

require 'sinatra/base'

class Session < Sinatra::Base
  get '/sessions/new' do
    slim 'session/new'.to_sym
  end

  post '/sessions' do
    some code here ...
  end
end

class Application < Sinatra::Base
  use Session

  get '/' do
    'hello, world!'
  end
end
{% endhighlight %}

这样一来，Session 就变成了 Application 的一个 middleware 了，具体如何实现的，之后再介绍。

#### 3. 通过 config.ru 实现 namespace.

一种很常见的需求，就是将应用的路由分成若干个命名空间，在 Rails 中可以用直接用 namespace 这个方法, 而在 Sinatra 的核心代码中没有这个方法，需要一个一个的写，比如:

{% highlight ruby %}
# api.rb

get '/api/users' do
end

get '/api/posts' do
end

get '/api/comments' do
end

...
{% endhighlight %}

太不环保了，这时我们可以使用 rackup 配置 config.ru:

{% highlight ruby %}
# app.rb

class Api < Sinatra::Base
  get '/' do
  end

  get '/users' do
  end

  get '/posts' do
  end

  get '/comments' do
  end
end

class Application < Sinatra::Base
  get '/' do
  end
end

# config.ru

require './app.rb'

map '/api' do
  run Api
end

run Application
{% endhighlight %}

轻松实现命名空间的功能。

#### 4. Mount on Rails Application

由于 Rails Application 也是一个 Rack 应用，所以将 Sinatra mount 到 Rails app 上面会非常无痛，只需要将你的 sinatra 打包成一个 gem 添加到 Rails Gemfile 里面, 然后在 Rails 应用目录下的 config.ru 配置一下就好了:

{% highlight ruby %}
# my_rails/config.ru

require ::File.expand_path('../config/environment',  __FILE__)

map '/my_sinatra' do
  run MySinatra::Application
end

run MyRails::Application
{% endhighlight %}

需要注意的是，为了不要让 rails 和 sinatra 的类名冲突了，通常会将 siantra 应用加个命名空间，就如同上面的 MySinatra 一样。
