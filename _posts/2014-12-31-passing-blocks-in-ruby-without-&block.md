---
layout: post
---

我们知道在 Ruby 的方法中有两种方式可以执行传入的代码块:

第一种是使用 yield 关键字:

```ruby
def say
  yield
end

say { 'Hello, World!' }

=> 'Hello, World!'
```

第二种是使用 & 符号定义参数:

```ruby
def say &block
  block.call
end

say { 'Hello, World!' }

=> 'Hello, World!'
```

由于第二种方式在调用时每次都会生成一个 Proc 对象, 在性能上要比第一种方式慢数倍, 所以尽量不要使用, 除非你确定需要一个 Proc 对象.
