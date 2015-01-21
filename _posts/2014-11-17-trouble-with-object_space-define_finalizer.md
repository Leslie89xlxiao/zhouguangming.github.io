---
layout: post
title: Trouble With ObjectSpace.define_finalizer
---

[ObjectSpace.define_finalizer](http://ruby-doc.org/core-2.1.4/ObjectSpace.html#method-c-define_finalizer) 方法可以接受一个 proc, 当某个对象被 GC 销毁时会回调该 proc. 当我按照文档使用这个方法的时候却发现一个很奇怪的现象, 我模仿官方写了如下的测试代码:

```ruby
def foo
  str = 'hello, world' * 1_000_000

  ObjectSpace.define_finalizer(str, proc { |id| puts "removing #{id}" })
end

100.times do |i|
  puts i
  foo
  sleep 1
end
```

原本预想的结果是: 程序运行过程中 GC 会时不时的销毁对象, 从而使得终端打印出 'removing object 19199560' 这样字符串出来, 并且该进程的内存占用应该是在一个小范围上下浮动.

实际发生的结果是: 程序运行过程中 GC 并没有销毁对象, 直到结束后终端连续打印出 'removing object 19199560' 这样的字符串, 并且该进程的内存占用持续上涨.

经过研究, 原来产生这个现象的原因是, finalize proc 的代码块是一个闭包, 其保存了当前运行环境(foo 方法)的上下文, 也包括 str 这个变量, 所以导致该字符串一直被引用, 从而无法被 GC 回收.

知道了原因之后, 便可以从 2 个方面修改上述代码, 使其按照我预想(正确)的方式运行:

* 手动更改 str = nil, 使得 'hello, world' * 1\_000\_000 这个字符串没有变量引用它

```ruby
def foo
  str = 'hello, world' * 1_000_000

  ObjectSpace.define_finalizer(str, proc { |id| puts "removing #{id}" })

  str = nil
end

100.times do |i|
  puts i
  foo
  sleep 1
end
```

* 不直接使用 proc, 用其他方式代替它, 避免 str 被保留在一个闭包中

```ruby
class Finalizer
  def self.finalize
    proc { |id| puts "removing #{id}" }
  end
end

def foo
  str = 'hello, world' * 1_000_000

  ObjectSpace.define_finalizer(str, Finalizer.finalizer)
end

100.times do |i|
  puts i
  foo
  sleep 1
end
```

一般来说第二种方式更合适, 标准库中 [lib/tempfile.rb](https://github.com/ruby/ruby/blob/trunk/lib/tempfile.rb#L131) 的实现便是一个很好的示例.
