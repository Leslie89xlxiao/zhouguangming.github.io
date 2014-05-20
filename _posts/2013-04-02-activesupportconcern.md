---
layout: post
title: "ActiveSupport::Concern"
description: ""
category:
tags: [ruby, rails]
---

ActiveSupport::Concern 在 Rails 源码中被大量使用，在 Rails 4.0 以后也会在 app 下面增加一个 concerns 的目录，用于管理你的 Module， 今天我会从源码的角度来看看这个东西是如何工作的。

<!--break-->

本来以为是个蛮复杂的东西，但是实际上一看，除了备注有用的代码只有 25 行（Rails 4.0)。

首先看看以前我们是怎么在 Rails 中用 Module 的：

{% highlight ruby %}
module M
  def included?(base)
    base.extend(ClassMethods)
    base.class_eval do
      callback_when_m_included
    end
  end

  def foo
    puts "foo"
  end

  module ClassMethods
    def bar
      puts "bar"
    end
  end
end
{% endhighlight %}

这是我个人比较钟爱的写法，非常容易理解，用 ClassMethod 把实例方法和类方法分离开，重写 included 方法，在 include 的时候，把 ClassMethod 的实例方法 extend 给 base 类, 看起来已经天衣无缝, 非常完美了。

使用 ActiveSupport::Concern 的写法为：

{% highlight ruby %}
require 'active_support/concern'

module M
  extend ActiveSupport::Concern

  included do
    callback_when_m_included
  end

  def foo
    puts "foo"
  end

  module ClassMethods
    def bar
      puts "bar"
    end
  end
end
{% endhighlight %}

看不出哪里好啊？难道就这样？带着好多疑问我打开了源码：

{% highlight ruby %}
module Concern
  def self.extended(base)
    base.instance_variable_set("@_dependencies", [])
  end

  def append_features(base)
    if base.instance_variable_defined?("@_dependencies")
      base.instance_variable_get("@_dependencies") << self
      return false
    else
      return false if base < self
      @_dependencies.each { |dep| base.send(:include, dep) }
      super
      base.extend const_get("ClassMethods") if const_defined?("ClassMethods")
      base.class_eval(&@_included_block) if instance_variable_defined?("@_included_block")
    end
  end

  def included(base = nil, &block)
    if base.nil?
      @_included_block = block
    else
      super
    end
  end
end
{% endhighlight %}

首先 ActiveSupport::Concern 重写了 self.extended(base) 方法，它的作用是当 Module M 调用 extend ActiveSupport::Concern 的时候为 M 创建一个实例变量 @\_dependencies 为 [], 然后 ActiveSupport::Concern 定义的实例方法 included 和 append_features 都变成了 M 的类方法, 导致 M 重写了 included 和 append_features 方法，这两个都是非常重要方法。
included 方法这段代码蛮巧妙的，我们知道一个 Module 在被 include 的时候会自动调用方法它的 included 类方法，并且会传入 include 它的类或者模块。但是当 M 主动去调用它的时候，没有传入 base，只是传入一个代码块，这时就发生了不同的情况，把这个代码块赋值给一个实例变量 @\_included_block，但是这个不会影响 M 正常的 included 回调。

接下来是 append_features 方法，和 included 类似，这个方法也是在 M 被 include 的时候被回调，并且传入 include 它的类或者模块，但是两个也不完全相同，因为 included 什么都没干，只是回调，而 append_features 却做了 include 背后真正的事情，所以当重写 append_features 的时候，或想还保留 include 真正的意义， 被需要使用 super 关键字。

M 被 ActiveSupport::Concern 重写的 self.append_features 主要作用是：首先 base （本例中的 C）没有定义实例变量 @\_dependencies, 其次 C 不是 M 的子类，所以接下来有四个步骤：

1. 把 @\_dependencies 分别被 C include，但是在此例中 M 的 @\_dependencies 为空，所以这步实际上没有意义。

2. 把 M 自己被 C include。

3. extend M 的 ClassMethods。

4. 执行代码块，调用 C 的方法。

若是没有 @\_dependencies， 我觉得和我的写法没有任何区别，最有意思的就是这个 @\_dependencies, 它使得模块之间依赖的处理变得更加优雅, 看看下面这个例子:

{% highlight ruby %}

module Foo
  def self.included(base)
    base.class_eval do
      def self.method_injected_by_foo
        ...
      end
    end
  end
end

module Bar
  def self.included(base)
    base.method_injected_by_foo
  end
end

class Host
  include Foo
  include Bar
end

{% endhighlight %}

这个时候想要 include Bar，必须也在要在 Host 中 include Foo，因为 Bar 中的 method\_injected\_by\_foo 依赖于 Foo， 但是实际上 Host 不该关心 Foo 是否被 include, 改为如下：

{% highlight ruby %}
module Bar
  include Foo

  def self.included(base)
    base.method_injected_by_foo
  end
end

class Host
  include Bar
end
{% endhighlight %}

这样做看上去不错，但是其实没有效果，因为 Foo 中的类方法被定义给了 Bar，include 不会把类方法一起给 Host, ActiveSupport::Concern 可以解决这个问题：

{% highlight ruby %}

require 'active_support/concern'

module Foo
  extend ActiveSupport::Concern

  included do
    class_eval do
      def self.method_injected_by_foo
        ...
      end
    end
  end
end

module Bar
  extend ActiveSupport::Concern
  include Foo

  included do
    self.method_injected_by_foo
  end
end

class Host
  include Bar
end
{% endhighlight %}

Bar include Foo 后，将 Foo 加入的 Bar 的 @\_dependencies 数组中，当 Host include Bar 的时候，实际上先 include Foo, 使得 Host 成为最终的那个 base, 这样做就可以把模块之间的依赖让模块自己去解决，而 Host 根本不用关心。
