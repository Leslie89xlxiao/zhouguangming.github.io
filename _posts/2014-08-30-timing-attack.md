---
layout: post
---

[计时攻击](http://en.wikipedia.org/wiki/Timing_attack) 属于旁路攻击的一种, 所谓旁路攻击就是通过对系统的物理学分析和实现方式分析, 而不是密码学分析或暴力破解, 来尝试破解密码学系统的行为. 密码学系统的电力消耗, 电磁波泄露, 时间差等信息都有可能提供对破解系统有帮助的信息.

而计时攻击就是利用时间差来对计算机进行攻击, 那么它的原理是什么? 我们拿 Rails 中的一段[代码](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/message_verifier.rb#L57)进行分析:


```ruby
# constant-time comparison algorithm to prevent timing attacks
def secure_compare(a, b)
  return false unless a.bytesize == b.bytesize

  l = a.unpack "C#{a.bytesize}"

  res = 0
  b.each_byte { |byte| res |= byte ^ l.shift }
  res == 0
end
```

这段代码的一个重要使用场景是验证签名 cookie 的有效性, 即外部传入的摘要与内部计算的摘要是否相等, 但是为什么不直接使用 `a == b`, 而是一个字符一个字符的比较, 并且看上去以一种十分低效的方式进行的?

原因正是为了预防计时攻击.

`a == b` 在 [Ruby](https://github.com/ruby/ruby/blob/trunk/string.c#L2461) 内部使用 [memcmp](http://man7.org/linux/man-pages/man3/memcmp.3.html) 函数进行比较, 当首次发现有两个字符不一致时, 便直接返回两个字符在 ASCII 码表中的差值. 在这样的机制下, CPU 运行的时间与字符串匹配度是有正关系的, 字符串匹配度越高, CPU 运行时间越长, 因此便可以通过对比时间差的方式逐一猜测破解.

预防计时攻击的方法通常也很简单, 使用 Constant-Time 的方式. 如上述比较摘要是否相等的示例代码, 当发现两个字符不一致时, 并没有立即返回, 而是继续比较下去, 因此使得计算时间不会变化.

虽然这样的处理, 时间复杂度提高了, 在语言级别的效率也降低了, 但在系统安全的关键部分, 付出了一点点性能的代价还是值得的.

也许你会觉得不解(我也是), 每个请求在不同的情况下包括网络延迟, 计算机运行状态都是不可能完全一致的, 况且这么一点点的时间差在 Rails 处理整个复杂的请求过程中(你应该知道, 在 Rails 中一个请求可能生成数以万计的 Ruby 对象)也显得非常的微不足道, 应该是非常难以利用的, 计时攻击难道仅仅存活在理论当中吗? 这篇[论文](http://www.cs.rice.edu/~dwallach/pub/crosby-timing2009.pdf)也许可以给我们带来一些参考.
