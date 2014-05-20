---
layout: post
title: "How sinatra works(1)"
description: ""
category:
tags: [ruby, sinatra]
---

一直觉得 sinatra 是个很棒的框架，小巧灵活，将 Ruby 的 DSL 能力发挥到了极致，其中有不少值得学习的东西，我将花一段时间研究这个框架，并记录一些比较重要的东西作为博客的一个系列。

<!--break-->

首先，使用 sinatra 开发一个最简单的 `hello, world!`:

{% highlight ruby %}
# app.rb

require 'sinatra'

get '/' do
  'hello, world!'
end

ruby app.rb
{% endhighlight %}

一个简单的 `hello, world!` 应用就完成了，接下来看看 sinatra 到底施了什么魔法, （sinatra 版本为 1.4.2）。

require sinatra 文件分别 requrie 了 sinatra/base 和 sinatra/mian, main 对象中的 get 方法由 Sinatra::Delegator 提供:

{% highlight ruby %}
# sinatra/base.rb line 1907

# Sinatra delegation mixin. Mixing this module into an object causes all
# methods to be delegated to the Sinatra::Application class. Used primarily
# at the top-level.
module Delegator #:nodoc:
  def self.delegate(*methods)
    methods.each do |method_name|
      define_method(method_name) do |*args, &block|
        return super(*args, &block) if respond_to? method_name
        Delegator.target.send(method_name, *args, &block)
      end
      private method_name
    end
  end

  delegate :get, :patch, :put, :post, :delete, :head, :options, :link, :unlink,
           :template, :layout, :before, :after, :error, :not_found, :configure,
           :set, :mime_type, :enable, :disable, :use, :development?, :test?,
           :production?, :helpers, :settings, :register

  class << self
    attr_accessor :target
  end

  self.target = Application
end
{% endhighlight %}

在 Delegator 这个 module 中定义实例方法，那么 mixin 这个 module，便可以使用这些方法，若宿主类或模块定义了该方法，则调用，否则将消息发给 Application 类。由于 sinatra/main.rb 文件在 main 中使用 extend Sinatra::Delegator，所以在 main 对象中可以使用 Delegator 中定义的实例方法了，这里之所以使用 extend 而不是 include，是因为 include 会把 Delegator 中的实例方法全部加给 Object 类，导致 Ruby 中的所有对象都具有这些方法了，并且将原来的方法覆盖，而 extend 只会让 main 对象拥有这些方法，而不是全部都 mixin 到 Object 中。

由此可见，sinatra 为了可以优美直接地使用 DSL，不惜将方法加入到 main / top-level 中。
