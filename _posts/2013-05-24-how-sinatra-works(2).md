---
layout: post
title: "How sinatra works(2)"
description: ""
category:
tags: [ruby, sinatra]
---

[上篇博客](/2013/05/22/how-sinatra-works(1\))介绍了 sinatra 如何将一些主要的方法加入到 main 对象，以达到优美直接的 DSL 的效果，今天将主要介绍一下 sinatra 的配置方法。

<!--break-->

我们知道 Rails 中专门有一个 Configuration 类来保存其配置变量, sinatra 却是大不相同。

sinatra 通过 Sinatra::Base 类中的 set 方法进行配置：

{% highlight ruby %}
# sinatra/base.rb line 1155

# Sets an option to the given value.  If the value is a proc,
# the proc will be called every time the option is accessed.
def set(option, value = (not_set = true), ignore_setter = false, &block)
  raise ArgumentError if block and !not_set
  value, not_set = block, false if block

  if not_set
    raise ArgumentError unless option.respond_to?(:each)
    option.each { |k,v| set(k, v) }
    return self
  end

  if respond_to?("#{option}=") and not ignore_setter
    return __send__("#{option}=", value)
  end

  setter = proc { |val| set option, val, true }
  getter = proc { value }

  case value
  when Proc
    getter = value
  when Symbol, Fixnum, FalseClass, TrueClass, NilClass
    getter = value.inspect
  when Hash
    setter = proc do |val|
      val = value.merge val if Hash === val
      set option, val, true
    end
  end

  define_singleton("#{option}=", setter) if setter
  define_singleton(option, getter) if getter
  define_singleton("#{option}?", "!!#{option}") unless method_defined? "#{option}?"
  self
end
{% endhighlight %}

该方法有四个参数，option, value, ignore_setter 和 &block。

option 为配置的名字，value 或 block 为配置的值，两者只能存在一个, 否则抛异常:

`set(:option, :value) { :value }`

如果传了只 block 进来，则将 value 的值设为 block，设置 not_set 为 false。

如果 value 和 block 都没传，那么 option 必须有 each 方法，然后遍历这个 option，递归的再分别调用 set 方法，结束:

`set(value: :option)` 或 `set([value1, value2])`

如果如果只传了 value 或者 一个 block，并且存在 option= 方法，并且 ignore_setter 为 false，那么调用原始的 option= 方法，结束。
{% highlight ruby %}
def option=(value)
  value
end

set(:option, :value)
{% endhighlight %}

若以上情况都没有发生: 设置 setter 为一个 proc 对象，该对象可以有一个参数 val ，调用 set(option, val, true); 设置 getter 为一个 proc 对象，返回 value 的值。

判断 value 的值，若是一个 proc， 则将 value 赋值给 getter; 若是 Symbol、Fixnum、 FalseClass、 TrueClass、 NilClass 则将 value.inspect 赋值给 getter; 若是一个哈希，重新赋值 setter 为另外一个 proc，该 proc 传入一个 val，如 val 是一个 hash，将 val 与 value 合并重新赋值给 val，调用 set(option, val, true)。


最后调用 define_singleton 方法:

{% highlight ruby %}
# sinatra/base.rb line 1319

# Dynamically defines a method on settings.
def define_singleton(name, content = Proc.new)
  # replace with call to singleton_class once we're 1.9 only
  (class << self; self; end).class_eval do
    undef_method(name) if method_defined? name
    String === content ? class_eval("def #{name}() #{content}; end") : define_method(name, &content)
  end
end
{% endhighlight %}

这段代码的作用是：该方法接受两个参数 name，和 content 打开 self 的单例类（在本例中为 Sinatra::Application 的单例类），如果 self 存在 name 的方法，则删除旧的方法; 判断 content 的类型，如是字符串，则使用 class_eval 方法后面跟字符串创建方法，否则直接使用 define_method 方法来为 self 创建方法。

假设调用 `set :interesting, true`, 通过 define_singleton 方法为 Sinatra::Application 创建了三个单例方法：

{% highlight ruby %}
def interesting
  true
end

def interesting?
  true
end

def interesting=(value)
  set(:interesting, value, true)
end
{% endhighlight %}

sinatra 的这种配置方式，和 Rails 完全不同，Rails 的配置都做为 Configuration 对象的实例变量, 而 sinatra 的配置完全就是定义方法。这样有一个明显的好处，就是不用创建很多实例变量，也不用再去抽象一个 Configuration 类，使得整个代码结构更紧凑, 这要得益于 Ruby 的魔法。但也是有缺点的，上述的代码量虽然不多，可涉及了不少的逻辑，也实现了很多功能, 因此理解起来可不太容易。
