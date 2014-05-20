---
layout: post
title: "How sinatra works(7)"
description: ""
category:
tags: [ruby, sinatra]
---

前面已经讲了好多关于 Sinatra 的一些实现，但是真正关键的有趣的地方还没说到，今天就说说请求进入到 Sinatra app 之后的过程, 从 Sinatra::Base#call! 开始吧。

<!--break-->

{% highlight ruby %}
# sinatra/base line 867

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

可以看到 Sinatra app 到 call! 这个方法之后发挥了它的主要功能: 解释 route，分析 request，构建 response。

首先是初始化几个实例变量: @env, @request, @response, @params. Sinatra::Request 和 Sinatra::Response 是从 Rack::Request 和 Rack::Response 继承而来的。@request.params 方法也是[继承](https://github.com/chneukirchen/rack/blob/master/lib/rack/request.rb#L223)过来的，是一个 hash，包括 GET 参数和 POST 参数, 下面是 indifferent_params 方法:

{% highlight ruby %}
# sinatra/base line 1041

# Enable string or symbol key access to the nested params hash.
def indifferent_params(object)
  case object
  when Hash
    new_hash = indifferent_hash
    object.each { |key, value| new_hash[key] = indifferent_params(value) }
    new_hash
  when Array
    object.map { |item| indifferent_params(item) }
  else
    object
  end
end

# Creates a Hash with indifferent access.
def indifferent_hash
  Hash.new {|hash,key| hash[key.to_s] if Symbol === key }
end
{% endhighlight %}

这段代码和 Rails 使用的 [HashWithIndifferentAccess](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/hash_with_indifferent_access.rb)功能有些类似，就是为 params 这个 Hash 提供不同的入口，但是 HashWithIndifferentAccess 提供了自定义的写，而这段代码并没有。


`template_cache.clear if settings.reload_templates` 这行代码作用是是否清除模板缓存，Sinatra 在开发环境每次都会清除模板缓存。

下面是 force_encoding 的源码:

{% highlight ruby %}
# sinatra/base line 1741

# Fixes encoding issues by
# * defaulting to UTF-8
# * casting params to Encoding.default_external
#
# The latter might not be necessary if Rack handles it one day.
# Keep an eye on Rack's LH #100.
def force_encoding(*args) settings.force_encoding(*args) end
if defined? Encoding
  def self.force_encoding(data, encoding = default_encoding)
    return if data == settings || data.is_a?(Tempfile)
    if data.respond_to? :force_encoding
      data.force_encoding(encoding).encode!
    elsif data.respond_to? :each_value
      data.each_value { |v| force_encoding(v, encoding) }
    elsif data.respond_to? :each
      data.each { |v| force_encoding(v, encoding) }
    end
    data
  end
else
  def self.force_encoding(data, *) data end
end
{% endhighlight %}

Sinatra 默认使用 utf-8 编码，这段代码就是将 params 转成 utf-8 编码。

一切准备就绪，进入到了解释 route 的环节。
