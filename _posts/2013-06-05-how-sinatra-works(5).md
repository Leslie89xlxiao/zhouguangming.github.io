---
layout: post
title: "How sinatra works(5)"
description: ""
category:
tags: [ruby, sinatra]
---

我已经在前面的一篇[文章](/2013/05/26/how-sinatra-works(3\)/)中详细的介绍了 Sinatra 是如何跑起来的了，这篇博客主要介绍一下 Sinatra Application 作为 Rack 的终点，如何相应请求的。

<!--break-->

我们已经知道，Rack 的规范非常简单，MiddleWare 响应两个方法，initialize 和 call，而终端的 application 最少只需要响应一个 call 方法就足够了，当我们把 Sinatra::Application 这个类作为整个 Rack 链的终点时，这个类也必须有一个 call 方法, 那接下来就从这个 call 方法入手吧。

{% highlight ruby %}
# sinatra/base.rb line 1609

def call(env)
  synchronize { prototype.call(env) }
end

# sinatra/base.rb line 1688
@@mutex = Mutex.new
def synchronize(&block)
  if lock?
    @@mutex.synchronize(&block)
  else
    yield
  end
end
{% endhighlight %}

Sinatra 默认是不加锁的，因为 sinatra 足够简单，完全可以保证本身是线程安全的，不像 Rails 代码量如此庞大，并且依赖了非常多的 gem，没法保证线程安全，所以 Rails 默认是不开启的（Rails 4.0 以后已经默认开启了）。如果你的代码中使用了非线程安全的 gem，那么通过 set :lock, true 的方式为每次请求加锁。

{% highlight ruby %}
# sinatra/base.rb line 1583

# The prototype instance used to process requests.
def prototype
  @prototype ||= new
end

# Create a new instance without middleware in front of it.
alias new! new unless method_defined? :new!

# Create a new instance of the class fronted by its middleware
# pipeline. The object is guaranteed to respond to #call but may not be
# an instance of the class new was called on.
def new(*args, &bk)
  instance = new!(*args, &bk)
  Wrapper.new(build(instance).to_app, instance)
end

# sinatra/base.rb line 855

def initialize(app = nil)
  super()
  @app = app
  @template_cache = Tilt::Cache.new
  yield self if block_given?
end
{% endhighlight %}

原始的 new 方法被 sinatra 别名成了 new! 新的 new! 方法首先实例化一个自己，也就是 Sinatra::Application, 在 Sinatra::Base 的 initilize 实例方法中，super 调用了 Templates 的 initilize 实例方法（目的不明），然后初始化两个实例变量，Tilt 是提供了多个模板引擎的接口，详情见[rtomayko / tilt](https://github.com/rtomayko/tilt)。接着构建 Rack 应用链:

{% highlight ruby %}
# sinatra/base.rb line 1599

# Creates a Rack::Builder instance with all the middleware set up and
# the given +app+ as end point.
def build(app)
  builder = Rack::Builder.new
  setup_default_middleware builder
  setup_middleware builder
  builder.run app
  builder
end
{% endhighlight %}

上面的代码需要对 [Rack::Builder](https://github.com/chneukirchen/rack/blob/master/lib/rack/builder.rb) 有一定的了解，Rack::Builder 提供了一些简单易用的接口用来构建 Rack 应用。Sinatra 所使用的 middleware 有两个部分，一部分是 sinatra 默认使用的，另一部分是用户自己所添加的。默认使用的部分也根据用户的配置选择是否加载，比如若用户想使用 session 的功能，需要配置: set :sessions, true, 或者使用 enable :sessions 方法，enable 方法就是把后面的参数全部设置成 true。用户自己需要使用的 middleware 可以使用 use middleware 的方式添加。关于 Sinatra 使用 middleware 的细节先放着，以后会专门有一篇介绍这些。最后返回一个 Wrapper 的实例。

{% highlight ruby %}
class Wrapper
  def initialize(stack, instance)
    @stack, @instance = stack, instance
  end

  def settings
    @instance.settings
  end

  def helpers
    @instance
  end

  def call(env)
    @stack.call(env)
  end

  def inspect
    "#<#{@instance.class} app_file=#{settings.app_file.inspect}>"
  end
end
{% endhighlight %}

这个 Wrapper 的作用是把 Sinatra app 和 Rack app 分割开来, 然后就没有然后了。

当一个请求进来以后，Rack app 被一层一层的执行过去，最后到了 Sinatra::Application 的实例这里:

{% highlight ruby %}
# Rack call interface.
def call(env)
  dup.call!(env)
end

attr_accessor :env, :request, :response, :params

def call!(env) # :nodoc:
  @env      = env
  @request  = Request.new(env)
  @response = Response.new
  @params   = indifferent_params(@request.params)
  template_cache.clear if settings.reload_templates
  force_encoding(@params)

  @response['Content-Type'] = nil
  invoke { dispatch! }
  invoke { error_block!(response.status) } unless @env['sinatra.error']

  unless @response['Content-Type']
    if Array === body and body[0].respond_to? :content_type
      content_type body[0].content_type
    else
      content_type :html
    end
  end

  @response.finish
end
{% endhighlight %}

为了在并发情况下各个参数互不影响，所以每一次都会 dup 一个新的实例调用 call! 方法。前面所说的都是准备工作，接下来才是奇迹发生的时刻，因为 Sinatra 的大部分工作都在这里发生, 内容很多，下次再说。
