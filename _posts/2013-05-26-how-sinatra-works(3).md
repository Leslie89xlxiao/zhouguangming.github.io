---
layout: post
title: "How sinatra works(3)"
description: ""
category:
tags: [ruby, sinatra]
---
[上一篇博客](/2013/05/24/how-sinatra-works(2\)) 讲到 sinatra 的配置方式, 今天介绍一下 sinatra 是怎么启动的。

我们知道 Sinatra 基于 Rack，所以它也应该遵循了 Rack 的相关规范(‘相关’真是个好词，用了它可以少解释好多东西，相关部门经常这么做)。由于 'hello, world' 应用使用的是 sinatra 的 classic style (另外还有 modular style，之后会介绍），所以 Sinatra::Application 作为 rack 的终点 application。Sinatra::Application 继承自 Sinatra::Base

<!--break-->

{% highlight ruby %}
# sinatra/base.rb line 1886

# Execution context for classic style (top-level) applications. All
# DSL methods executed on main are delegated to this class.
#
# The Application class should not be subclassed, unless you want to
# inherit all settings, routes, handlers, and error pages from the
# top-level. Subclassing Sinatra::Base is highly recommended for
# modular applications.
class Application < Base
  set :logging, Proc.new { ! test? }
  set :method_override, true
  set :run, Proc.new { ! test? }
  set :session_secret, Proc.new { super() unless development? }
  set :app_file, nil

  def self.register(*extensions, &block) #:nodoc:
    added_methods = extensions.map {|m| m.public_instance_methods }.flatten
    Delegator.delegate(*added_methods)
    super(*extensions, &block)
  end
end
{% endhighlight %}

Base 类有一个 inherited 的回调：

{% highlight ruby %}
# sinatra/base line 1682

def inherited(subclass)
  subclass.reset!
  subclass.set :app_file, caller_files.first unless subclass.app_file?
  super
end

# sinatra/base line 1121

def reset!
  @conditions     = []
  @routes         = {}
  @filters        = {:before => [], :after => []}
  @errors         = {}
  @middleware     = []
  @prototype      = nil
  @extensions     = []

  if superclass.respond_to?(:templates)
    @templates = Hash.new { |hash,key| superclass.templates[key] }
  else
    @templates = {}
  end
end

# sinatra/base line 1715

# Like Kernel#caller but excluding certain magic entries and without
# line / method information; the resulting array contains filenames only.
def caller_files
 cleaned_caller(1).flatten
end

# sinatra/base line 1733

# Like Kernel#caller but excluding certain magic entries
def cleaned_caller(keep = 3)
  caller(1).
    map    { |line| line.split(/:(?=\d|in )/, 3)[0,keep] }.
    reject { |file, *_| CALLERS_TO_IGNORE.any? { |pattern| file =~ pattern } }
end

# sinatra/base line 1698

CALLERS_TO_IGNORE = [ # :nodoc:
  /\/sinatra(\/(base|main|showexceptions))?\.rb$/,    # all sinatra code
  /lib\/tilt.*\.rb$/,                                 # all tilt code
  /^\(.*\)$/,                                         # generated code
  /rubygems\/(custom|core_ext\/kernel)_require\.rb$/, # rubygems require hacks
  /active_support/,                                   # active_support require hacks
  /bundler(\/runtime)?\.rb/,                          # bundler require hacks
  /<internal:/,                                       # internal in ruby >= 1.9.2
  /src\/kernel\/bootstrap\/[A-Z]/                     # maglev kernel files
]
{% endhighlight %}

上述代码有两个作用：

1. 初始化 Sinatra::Application 类的实例变量

2. 设置 app_file 为 app.rb

caller_files 方法是对 caller 做了一些特殊处理, app_file 是调用堆栈的最后一个用户文件。

inherited 回调之后，Application 又设置了一些值，值得注意的是，这边又把 app_file 设置为 nil 了。

sinatra/base.rb 结束之后，回到 sinatra/main 中：

{% highlight ruby %}
module Sinatra
  class Application < Base
    # we assume that the first file that requires 'sinatra' is the
    # app_file. all other path related options are calculated based
    # on this path by default.
    set :app_file, caller_files.first || $0

    set :run, Proc.new { File.expand_path($0) == File.expand_path(app_file) }

    if run? && ARGV.any?
      require 'optparse'
      OptionParser.new { |op|
        op.on('-p port',   'set the port (default is 4567)')                { |val| set :port, Integer(val) }
        op.on('-o addr',   "set the host (default is #{bind})")             { |val| set :bind, val }
        op.on('-e env',    'set the environment (default is development)')  { |val| set :environment, val.to_sym }
        op.on('-s server', 'specify rack server/handler (default is thin)') { |val| set :server, val }
        op.on('-x',        'turn on the mutex lock (default is off)')       {       set :lock, true }
      }.parse!(ARGV.dup)
    end
  end

  at_exit { Application.run! if $!.nil? && Application.run? }
end
{% endhighlight %}

这边的代码主要是为了兼容两种启动方式，因为前面说过，sinatra 基于 rack，那么当然可以用 rackup 启动, 但是这里先不讨论这种情况。

设置 app_file 又为 'app.rb', 设置 run, 是一个 proc，每次调用都会运行 proc 里面的内容，由于 $0 和 app_file 是同样的值(sinatra 用了很多类似于 $0 这样的全局变量，我在[这篇博客](/2013/04/20/ruby-default-global-variables/)里做了一些总结), 所以在运行 ruby app.rb 的时候，后面可以跟一些参数, 比如配置端口、地址之类的。

最后在这个文件即将结束的时候，出现了 at_exit 回调，这个方法在进程结束前执行，内容是 Application.run!, 目测 sinatra 要跑起来了。

在 sinatra/base 里面找到 run! 方法:

{% highlight ruby %}
# sinatra/base line 1561

# Run the Sinatra app as a self-hosted server using
# Thin, Puma, Mongrel, or WEBrick (in that order). If given a block, will call
# with the constructed handler once we have taken the stage.
def run!(options = {})
  set options
  handler         = detect_rack_handler
  handler_name    = handler.name.gsub(/.*::/, '')
  server_settings = settings.respond_to?(:server_settings) ? settings.server_settings : {}
  handler.run self, server_settings.merge(:Port => port, :Host => bind) do |server|
    unless handler_name =~ /cgi/i
      $stderr.puts "== Sinatra/#{Sinatra::VERSION} has taken the stage " +
      "on #{port} for #{environment} with backup from #{handler_name}"
    end
    [:INT, :TERM].each { |sig| trap(sig) { quit!(server, handler_name) } }
    server.threaded = settings.threaded if server.respond_to? :threaded=
    set :running, true
    yield server if block_given?
  end
rescue Errno::EADDRINUSE
  $stderr.puts "== Someone is already performing on port #{port}!"
end
{% endhighlight %}

了解 rack 基础的人我想对上面这段代码是不会陌生的，这就是 rack 提供的接口，sinatra 直接拿来用了, 如此一来，一个简单的 sinatra 应用就真的跑起来了。
