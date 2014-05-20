---
layout: post
title: "How sinatra works(8)"
description: ""
category:
tags: [ruby, sinatra]
---

上篇博客分析 call! 方法分析了一半，现在进入到了 invoke { dispatch! } 这个方法中:

<!--break-->

{% highlight ruby %}
# sinatra/base line 1033

# Run the block with 'throw :halt' support and apply result to the response.
def invoke
  res = catch(:halt) { yield }
  res = [res] if Fixnum === res or String === res
  if Array === res and Fixnum === res.first
    res = res.dup
    status(res.shift)
    body(res.pop)
    headers(*res)
  elsif res.respond_to? :each
    body res
  end
  nil # avoid double setting the same response tuple twice
end
{% endhighlight %}

这段代码有一个比较不常见的，并且我个人认为非常有趣重要的知识点，就是 catch throw. 在其他语言中 try catch throw 通常用于异常机制，但是在 ruby 中，catch 与 throw 则用来做逻辑控制: 在 catch 后面的代码块内不管任何语句只要调用 throw(:halt) 后，代码都会返回到 catch 的语句。不同于 return 和 break，return 通常是方法的返回，或者是在 lambda 中返回, break 通常用于结束代码块。而 catch 后面的代码无论经过多少的方法调用，或者 block call，只要在其中任意的地方出现 throw(:halt)，可以是别的方法内，也可以是其它任何的 proc 或 代码块， 都会回到 catch 关键字去。SInatra 这样做的原因是你可以在任何时候，完成 response 的构造。

{% highlight ruby %}
# sinatra/base line 1048

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

在 Sinatra 中静态资源，例如图片，js，css 等等默认在设置放在与 app.rb 同级的 public 目录下，这也是国际惯例。如果你的请求是 GET 或者 HEAD 请求，并且存在 public 这个目录，那么去看看你的请求是否是静态资源:

{% highlight ruby %}
# sinatra/base line 1000

# Attempt to serve static files from public directory. Throws :halt when
# a matching file is found, returns nil otherwise.
def static!
  return if (public_dir = settings.public_folder).nil?
  public_dir = File.expand_path(public_dir)

  path = File.expand_path(public_dir + unescape(request.path_info))
  return unless path.start_with?(public_dir) and File.file?(path)

  env['sinatra.static_file'] = path
  cache_control(*settings.static_cache_control) if settings.static_cache_control?
  send_file path, :disposition => nil
end
{% endhighlight %}

首先将请求路径解码，unescape 方法定义在 Rack::Utils 里面，Sinatra::Base 有 include 它。若在 public_dir 下面找到响应的文件，那么就认为该请求是一个静态请求，根据配置情况是否设置 response header 的 Cache-Control。Sinatra 默认开发环境设置该值为 false，也就是不让浏览器缓存住他们，以方便调试。cache_control 方法如下:

{% highlight ruby %}
# sinatra/base line 430

# Specify response freshness policy for HTTP caches (Cache-Control header).
# Any number of non-value directives (:public, :private, :no_cache,
# :no_store, :must_revalidate, :proxy_revalidate) may be passed along with
# a Hash of value directives (:max_age, :min_stale, :s_max_age).
#
#   cache_control :public, :must_revalidate, :max_age => 60
#   => Cache-Control: public, must-revalidate, max-age=60
#
# See RFC 2616 / 14.9 for more on standard cache control directives:
# http://tools.ietf.org/html/rfc2616#section-14.9.1
def cache_control(*values)
  if values.last.kind_of?(Hash)
    hash = values.pop
    hash.reject! { |k,v| v == false }
    hash.reject! { |k,v| values << k if v == true }
  else
    hash = {}
  end

  values.map! { |value| value.to_s.tr('_','-') }
  hash.each do |key, value|
    key = key.to_s.tr('_', '-')
    value = value.to_i if key == "max-age"
    values << [key, value].join('=')
  end

  response['Cache-Control'] = values.join(', ') if values.any?
end
{% endhighlight %}

最后调用 send_file 方法:

{% highlight ruby %}
# sinatra/base line 342

# Use the contents of the file at +path+ as the response body.
def send_file(path, opts = {})
  if opts[:type] or not response['Content-Type']
    content_type opts[:type] || File.extname(path), :default => 'application/octet-stream'
  end

  disposition = opts[:disposition]
  filename    = opts[:filename]
  disposition = 'attachment' if disposition.nil? and filename
  filename    = path         if filename.nil?
  attachment(filename, disposition) if disposition

  last_modified opts[:last_modified] if opts[:last_modified]

  file      = Rack::File.new nil
  file.path = path
  result    = file.serving env
  result[1].each { |k,v| headers[k] ||= v }
  headers['Content-Length'] = result[1]['Content-Length']
  halt opts[:status] || result[0], result[2]
rescue Errno::ENOENT
  not_found
end
{% endhighlight %}

上面代码可以主要用到了 [Rack::File](https://github.com/chneukirchen/rack/blob/master/lib/rack/file.rb) 这个类来解析图片信息, 然后将结果设置为相应的 header。Rack::File#serving 方法会判断静态资源是否被修改过，若没有，被修改过，则返回的状态码为 304, 也就是 Not Modify。最后调用 halt 方法:

{% highlight ruby %}
# sinatra/base line 908

# Exit the current block, halts any further processing
# of the request, and returns the specified response.
def halt(*response)
  response = response.first if response.length == 1
  throw :halt, response
end
{% endhighlight %}

halt 方法主要是用来设置 response 的 status 并且执行 throw :halt, 还记得它吧，它会让逻辑回到外部第一个 catch(:halt) 的地方, 并且把第二参数作为返回值。这里会返回到第一个 invoke 的地方，invoke 方法接下去会设置 response 的 status，header，和 body, 接着回到了 call! 中 invoke 的地方，因为第一次 invoke 返回了 nil，所以外面的 invoke 不会重复第一次的操作。由于 @response['Content-Type'] 已经在设置过了，所以下面不会再重新设置, 最后调用 @response.finish

{% highlight ruby %}
# sinatra/base line 128

def finish
  result = body

  if drop_content_info?
    headers.delete "Content-Length"
    headers.delete "Content-Type"
  end

  if drop_body?
    close
    result = []
  end

  if calculate_content_length?
    # if some other code has already set Content-Length, don't muck with it
    # currently, this would be the static file-handler
    headers["Content-Length"] = body.inject(0) { |l, p| l + Rack::Utils.bytesize(p) }.to_s
  end

  [status.to_i, headers, result]
end
{% endhighlight %}

这段代码就是为 response 做一些善后的工作，比如算 content length 之类的，中间那个 close 方法是为什么不太明白。最后组装成 Rack 标准的返回值。

但此时 result 是一个 Rack::File 实例，不可能直接返回给客户端的。后面的工作就是 Rack 做的了，当请求从 Sinatra 出去后，Rack Handler 还会对 Response 做最后的 ‘包装’，这其中就包括读去静态文件作为 body 而不是一个 Rack::File 的对象传给客户端。
