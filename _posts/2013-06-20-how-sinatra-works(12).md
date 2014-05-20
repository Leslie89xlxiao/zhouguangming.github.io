---
layout: post
title: "How sinatra works(12)"
description: ""
category:
tags: [ruby, sinatra]
---

当 Sinatra 在请求过程中出现异常时，会捕获这个异常，并且调用 handle_exception! 方法:

<!--break-->

{% highlight ruby %}
# sinatra/base.rb line 1071

# Error handling during requests.
def handle_exception!(boom)
  @env['sinatra.error'] = boom

  if boom.respond_to? :http_status
    status(boom.http_status)
  elsif settings.use_code? and boom.respond_to? :code and boom.code.between? 400, 599
    status(boom.code)
  else
    status(500)
  end

  status(500) unless status.between? 400, 599

  if server_error?
    dump_errors! boom if settings.dump_errors?
    raise boom if settings.show_exceptions? and settings.show_exceptions != :after_handler
  end

  if not_found?
    headers['X-Cascade'] = 'pass' if settings.x_cascade?
    body '<h1>Not Found</h1>'
  end

  res = error_block!(boom.class, boom) || error_block!(status, boom)
  return res if res or not server_error?
  raise boom if settings.raise_errors? or settings.show_exceptions?
  error_block! Exception, boom
end
{% endhighlight %}

boom 是异常的类，比如没有找到匹配路由的 NotFound:

{% highlight ruby %}
# sinatra/base.rb line 217

class NotFound < NameError #:nodoc:
  def http_status; 404 end
end

{% endhighlight %}

该类定义了一个 http_status 实例方法.

handle_exception! 方法首先把 boom 放到 sinatra.error 环境变量里面，然后设置 response 的 status。

若错误是一个服务器端异常(即 http status 在 500 - 599 之间），判断 settings.dump_errors?(开发环境默认为 true)，是否调用 dump_errors! 方法:

{% highlight ruby %}
# sinatra/base.rb line 1117

def dump_errors!(boom)
  msg = ["#{boom.class} - #{boom.message}:", *boom.backtrace].join("\n\t")
  @env['rack.errors'].puts(msg)
end
{% endhighlight %}

判断 settings.show_exceptions(开发环境默认为 true) 是否执行 raise boom, 若执行了 raise boom，则将错误信息交给 Sinatra::ShowExceptions 来处理。

若错误是一个 404 错误，判断 settings.x_cascade?(默认为 true） 是否设置 'X-Cascade' header.

设置 response body 为 `<h1>Not Found</h1>`

Sinatra 可以使用 Sinatra::Base.error 自定义异常:

{% highlight ruby %}
# sinatra/base.rb line 1225

# Define a custom error handler. Optionally takes either an Exception
# class, or an HTTP status code to specify which errors should be
# handled.
def error(*codes, &block)
  args  = compile! "ERROR", //, block
  codes = codes.map { |c| Array(c) }.flatten
  codes << Exception if codes.empty?
  codes.each { |c| (@errors[c] ||= []) << args }
end
{% endhighlight %}

首先通过 compile! 方法组装 error, 就像组装 filter 和 route 那样，然后放到 @errors 这个 Sinatra::Base 的类实例变量里面。

Sinatra 会默认设置一些异常信息, 如 500:

{% highlight ruby %}
# sinatra/base.rb line 1829

error ::Exception do
  response.status = 500
  content_type 'text/html'
  '<h1>Internal Server Error</h1>'
end
{% endhighlight %}

以及 development 环境的 404:

{% highlight ruby %}
# sinatra/base.rb line 1835

configure :development do
  get '/__sinatra__/:image.png' do
    filename = File.dirname(__FILE__) + "/images/#{params[:image]}.png"
    content_type :png
    send_file filename
  end

  error NotFound do
    content_type 'text/html'

    if self.class == Sinatra::Application
      code = <<-RUBY.gsub(/^ {12}/, '')
        #{request.request_method.downcase} '#{request.path_info}' do
          "Hello World"
        end
      RUBY
    else
      code = <<-RUBY.gsub(/^ {12}/, '')
        class #{self.class}
          #{request.request_method.downcase} '#{request.path_info}' do
            "Hello World"
          end
        end
      RUBY

      file = settings.app_file.to_s.sub(settings.root.to_s, '').sub(/^\//, '')
      code = "# in #{file}\n#{code}" unless file.empty?
    end

    (<<-HTML).gsub(/^ {10}/, '')
      <!DOCTYPE html>
      <html>
      <head>
        <style type="text/css">
        body { text-align:center;font-family:helvetica,arial;font-size:22px;
          color:#888;margin:20px}
        #c {margin:0 auto;width:500px;text-align:left}
        </style>
      </head>
      <body>
        <h2>Sinatra doesn&rsquo;t know this ditty.</h2>
        <img src='#{uri "/__sinatra__/404.png"}'>
        <div id="c">
          Try this:
          <pre>#{code}</pre>
        </div>
      </body>
      </html>
    HTML
  end
end
{% endhighlight %}

定义好 error 后可以通过 error_block 方法, 来找到对应的 error:

{% highlight ruby %}
# sinatra/base.rb line 1101

# Find an custom error block for the key(s) specified.
def error_block!(key, *block_params)
  base = settings
  while base.respond_to?(:errors)
    next base = base.superclass unless args_array = base.errors[key]
    args_array.reverse_each do |args|
      first = args == args_array.first
      args += [block_params]
      resp = process_route(*args)
      return resp unless resp.nil? && !first
    end
  end
  return false unless key.respond_to? :superclass and key.superclass < Exception
  error_block!(key.superclass, *block_params)
end
{% endhighlight %}

若在程序中没有代码上产生异常，而是自己定义的错误状态，在 call! 方法的 dispatch! 之后，还是会调用以此 error_block! 方法，根据 http status 来查找是否有对应的 error 定义。
