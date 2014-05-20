---
layout: post
title: "Add support for client caching via ETags"
description: ""
category: 
tags: [rails, rack, http]
---
###ETag

> HTTP 协议规格说明定义 ETag 为“被请求变量的实体值”。另一种说法是，ETag 是一个可以与Web资源关联的记号 (token)。典型的Web资源可以一个 Web 页，但也可能是 JSON 或 XML 文档。服务器单独负责判断记号是什么及其含义，并在 HTTP 响应头中将其传送到客户端，以下是服务器端返回的格式：

<!--break-->

> ETag: "7c57d5ec893a3bdf516ec5d61415fd6a" 

> 客户端的查询更新格式是这样的：

> If-None-Match : "7c57d5ec893a3bdf516ec5d61415fd6a" 

> 如果ETag没改变，则返回状态 304，这也和 Last-Modified 一样。

使用 ETag 提高性能的原理很简单，已一个博客为例，假设博客是一个简单的 Rails 程序(事实上它是 Jekyll)，因为我个人不会每天都更新这个博客，也就是说大多数时间读者看到的内容都是一样的，但是客户端每次都需要下载服务器端的返回的 body，这个是完全没有必要的。按照通常的方式，我们可以在服务器端为每篇博客做缓存，但即使这样，只是解决了服务器端的性能问题，假设一个读者是手机用户，并且在没有 wifi 的条件下，反复刷新这个博客，所产生的没必要的流量会让他觉得十分坑爹，因此还有一个通用的解决方案，就是使用客户端缓存。

我们知道现在的 HTTP 客户端基本上都能支持缓存，比如浏览器 和 IOS 的应用，怎么有效利用它们呢？今天我将介绍通过 ETag 来利用客户端缓存。

还是以我这个博客为例，每一篇文章都是一个资源，我们为这个资源标记唯一标识，通常是这个资源的 id，这是相对于其它的文章或资源，还要有一个状态标识，很自然的想到资源的 updated_at 字段，因为每次更新资源都会修改这个时间戳，通过这两个标识合并起来的结果来表示一个资源在某一个时间的状态，将这个状态进行签名得到的值就可以作为 ETag 了，在第一次请求时，服务器端将计算出 ETag 的值，放在 Response 的 Header 中发给客户端，客户端收到这个 ETag，第二次请求的时候就把它放在 Request 的 Header 中发回来，服务器通过对比自己产生的 ETag 和 收到的 ETag，如果是一样的，就可以认为客户端请求的东西没有变化，返回 304 状态码，客户端收到 304 后，就可以放心的用自己本地存储的缓存了。要是我在这过程中修改了这篇文章的内容，会导致 updated_at 这个字段发生变化，从而使得服务器端计算出来的 ETag 结果发生变化，对比不成功后，返回 200 状态码和新的 ETag，周而复始。

大概了解了一下原理后，接下来看看 Rails 在代码上是怎么做的，Etag 在 Rails 中是默认就支持的，不管在什么环境都已经开启了，通过 rake middleware 返回的结果可以看到支持 ETag 的 middleware。在 action 里面可以使用 stale? 和 fresh_when 方法:

{% highlight ruby %}
class PostsController < ApplicationController
  def show
    @post = Post.find params[:id]
    fresh_when :etag => @post
  end
end
{% endhighlight %}

这样，Response 的 ETag 就和 @post 这个对象有关了，并且每次 Request 进来都会对比，要是不调用这个方法，服务器也会根据 Rack::ETag 这个 middleware 产生 ETag， 这个 ETag 由 Response 的 body 产生的, 但这里我有个疑问， __这个 ETag 产生的作用是什么？__ 因为没有地方可以用到。
下面看看 fresh_when 这个方法的具体实现:

{% highlight ruby %}
# actionpack-3.2.12/lib/action_controller/metal/conditional_get.rb line 39

def fresh_when(record_or_options, additional_options = {})
  if record_or_options.is_a? Hash
    options = record_or_options
    options.assert_valid_keys(:etag, :last_modified, :public)
  else
    record  = record_or_options
    options = { :etag => record, :last_modified => record.try(:updated_at) }.merge(additional_options)
  end

  response.etag          = options[:etag]          if options[:etag]
  response.last_modified = options[:last_modified] if options[:last_modified]
  response.cache_control[:public] = true if options[:public]

  head :not_modified if request.fresh?(response)
end
{% endhighlight %}

代码不复杂，首先把要生成 ETag 的对象传进来，也就是上例中的 @post，response.etag= 方法如下： 

{% highlight ruby %}
# action_dispatch-3.2.12/lib/action_dispatch/http/cache.rb line 63

def etag=(etag)
  key = ActiveSupport::Cache.expand_cache_key(etag)
  @etag = self[ETAG] = %("#{Digest::MD5.hexdigest(key)}")
end
{% endhighlight %}

ActiveSupport::Cache.expand_cache_key(etag) 方法在本例中实际上就是调用 @post.cache_key.to_s, 得到的是类似于 "posts/1-20130314082714", 这个字符串的含义是： posts 代表资源集合的名字，1 代表资源 id，后面的数字代表这个资源 updated_at 的时间戳，即和我上面分析的一样用这三样表示一个资源的状态标识。

除此之外还可以看到，使用 fresh_when 方法 header 中还会带上 last_modified。

request.fresh? 方法也就是上面所说的比较的过程，代码如下：

{% highlight ruby %}
# action_dispatch-3.2.12/lib/action_dispatch/http/cache.rb line 28

def fresh?(response)
  last_modified = if_modified_since
  etag          = if_none_match

  return false unless last_modified || etag

  success = true
  success &&= not_modified?(response.last_modified) if last_modified
  success &&= etag_matches?(response.etag) if etag
  success
end
{% endhighlight %}

使 response.last_modified 和 request header 的 If-Modified-Since 比较，使 response.etag 和 request header 的 If-None-Match 比较，只要有一个存在，并且比较没问题，则返回 true。最后 head :not_modified 设置 response 的 status 为 304。当然若 fresh 返回 false，则继续向下执行，渲染 View，返回 200，然后使客户端更新成新的 ETag 与 Last Modified。

简单的介绍就是这样，当然很多细节就不再多介绍，比如 stale? 方法的使用，比如多个对象的情况, 比如没有 updated_at 字段怎么办，还有或者设置 [cache-control](http://condor.depaul.edu/dmumaugh/readings/handouts/SE435/HTTP/node24.html) 为 public 等等，大家有兴趣可以去看看源码。

最后若谁能解答我上面的 __疑问__ ，希望能发我[邮件](mailto:zhouguangming@gmail.com)给我。


