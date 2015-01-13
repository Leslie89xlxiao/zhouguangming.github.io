---
layout: post
---

上周看了一下 MRI 中 Timeout 的实现, 原理比较简单, [lib/timeout](https://github.com/ruby/ruby/blob/v2_2_0_rc1/lib/timeout.rb) 中就 100 多行, 核心代码如下:

```ruby
bl = proc do |exception|
  begin
    x = Thread.current
    y = Thread.start {
      begin
        sleep sec
      rescue => e
        x.raise e
      else
        x.raise exception, message
      end
    }
    return yield(sec)
  ensure
    if y
      y.kill
      y.join # make sure y is dead.
    end
  end
end
```

1. 将当前线程赋值给 x

2. 创建一个线程 y, sleep n 秒, n 为 timeout 的时长; 执行 x 中 timeout 传入的 block

3. 若 x 的 block 先结束, 则使用 Thread#kill 杀死 y 线程

4. 若 y 的 sleep 先结束, 则使用 Thread#raise 使 x 抛出异常

以上内容只是在 Ruby 层面分析了一下 Timeout 的实现, 比较浅显, 希望了解更多的同学可以好好看看[这篇文章](http://redgetan.cc/understanding-timeouts-in-cruby/), 相信会给你醍醐灌顶的感觉.
