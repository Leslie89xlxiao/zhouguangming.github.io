---
layout: post
title: "How sinatra works(11)"
description: ""
category:
tags: [ruby, sinatra]
---

当我们在 sinatra 中定义了 filter 和 route 之后，若某次请求不是静态资源的请求，那么紧接着会尝试调用 filter!(:before) 方法:

<!--break-->

{% highlight ruby %}
# sinatra/base.rb line 938

# Run filters defined on the class and all superclasses.
def filter!(type, base = settings)
  filter! type, base.superclass if base.superclass.respond_to?(:filters)
  base.filters[type].each { |args| process_route(*args) }
end
{% endhighlight %}

执行某个类的 filter 时，也要执行其父类的 filters, 这样可以实现只需要在父类中生成 filter，子类继承这个父类就可以了，就像在 Rails 中 Admin::BaseController 中定义一个 filter，其它类如 Admin::PostsController 继承它就可以了。

接下来遍历所有的 before 类型的 filter，并且执行 process_route 方法:

{% highlight ruby %}
# sinatra/base.rb line 969

# If the current request matches pattern and conditions, fill params
# with keys and call the given block.
# Revert params afterwards.
#
# Returns pass block.
def process_route(pattern, keys, conditions, block = nil, values = [])
  route = @request.path_info
  route = '/' if route.empty? and not settings.empty_path_info?
  return unless match = pattern.match(route)
  values += match.captures.to_a.map { |v| force_encoding URI.unescape(v) if v }

  if values.any?
    original, @params = params, params.merge('splat' => [], 'captures' => values)
    keys.zip(values) { |k,v| Array === @params[k] ? @params[k] << v : @params[k] = v if v }
  end

  catch(:pass) do
    conditions.each { |c| throw :pass if c.bind(self).call == false }
    block ? block[self, values] : yield(self, values)
  end
ensure
  @params = original if original
end
{% endhighlight %}

前面已经猜测过 sinatra 是如何处理请求的了: 首先判断 request 的 path 是否与正则匹配，若匹配，则抽取出包含在 path 中的参数，然后判断 conditions 是否完全符合，如符合则执行代码块，最后把参数回溯为初始化。

执行完 filter! 之后执行 route! 方法:

{% highlight ruby %}
# sinatra/base.rb line 944

# Run routes defined on the class and all superclasses.
def route!(base = settings, pass_block = nil)
  if routes = base.routes[@request.request_method]
    routes.each do |pattern, keys, conditions, block|
      pass_block = process_route(pattern, keys, conditions) do |*args|
        env['sinatra.route'] = block.instance_variable_get(:@route_name)
        route_eval { block[*args] }
      end
    end
  end

  # Run routes defined in superclass.
  if base.superclass.respond_to?(:routes)
    return route!(base.superclass, pass_block)
  end

  route_eval(&pass_block) if pass_block
  route_missing
end
{% endhighlight %}

首先是找到对应 HTTP verb 的 routes，然后遍历执行 process_route 方法，寻找出合适的 route，设置 sinatra.route 的 env 变量，最后执行 route_eval 方法:

{% highlight ruby %}
# sinatra/base.rb line 964

# Run a route block and throw :halt with the result.
def route_eval
  throw :halt, yield
end
{% endhighlight %}

这个方法的主要作用执行 route 中定义的 block，然后 throw :halt，也就是结束 route 的部分。若在当前类中没有找到对应的 route，则到父类中寻找, 若所有的类中都没有定义则执行 route_missing 方法:

{% highlight ruby %}
# sinatra/base.rb line 993

# No matching route was found or all routes passed. The default
# implementation is to forward the request downstream when running
# as middleware (@app is non-nil); when no downstream app is set, raise
# a NotFound exception. Subclasses can override this method to perform
# custom route miss logic.
def route_missing
  if @app
    forward
  else
    raise NotFound
  end
end

# sinatra/base.rb line 926

# Forward the request to the downstream app -- middleware only.
def forward
  fail "downstream app not set" unless @app.respond_to? :call
  status, headers, body = @app.call env
  @response.status = status
  @response.body = body
  @response.headers.merge! headers
  nil
end
{% endhighlight %}

route_missing 先查看是否有 @app 变量，如果 sinatra 作为 middleware, 那么肯定会有一个 @app，则继续像下执行, 否则作为 rack 的终点, 抛出 NotFound 的异常。之所以这边抛了异常却没有退出进程，是因为在 dispatch! 方法中会 rescue 这些异常:

{% highlight ruby %}
# sinatra/base.rb line 1054

# Dispatch a request with error handling.
def dispatch!
  invoke do
    static! if settings.static? && (request.get? || request.head?)
    filter! :before
    route!
  end
rescue ::Exception => boom
  invoke { handle_exception!(boom) }
ensure
  begin
    filter! :after unless env['sinatra.static_file']
  rescue ::Exception => boom
    invoke { handle_exception!(boom) } unless @env['sinatra.error']
  end
end
{% endhighlight %}

最后，执行 filter!(:after), 下次讲 Sinatra 如何处理异常。
