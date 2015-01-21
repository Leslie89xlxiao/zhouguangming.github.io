---
layout: post
title: SecureRandom in Ruby
---

[SecureRandom](http://www.ruby-doc.org/stdlib-2.2.0/libdoc/securerandom/rdoc/SecureRandom.html) 是 Ruby 中非常重要的库, 用于生成各类随机字符串, SecureRandom 中的核心方法是 `self.gen_random`, 其定义方式分三种情况:

1. 若系统安装了 OpenSSL , 则使用 [OpenSSL::Random](http://ruby-doc.org/stdlib-trunk/libdoc/openssl/rdoc/OpenSSL/Random.html#method-c-random_add) 来生成

2. 若系统为 Windows, 则使用 Windows API 中的 [CryptGenRandom](http://en.wikipedia.org/wiki/CryptGenRandom) 来生成

3. 若系统为类 Unix, 则使用 [/dev/urandom](http://en.wikipedia.org/wiki/dev/random) 字符设备来生成

下面我们看一下如何使用 /dev/urandom 来生成:

```ruby
def self.gen_random(n)
  flags = File::RDONLY
  flags |= File::NONBLOCK if defined? File::NONBLOCK
  flags |= File::NOCTTY if defined? File::NOCTTY
  begin
    File.open("/dev/urandom", flags) {|f|
      unless f.stat.chardev?
        break
      end
      ret = f.read(n)
      unless ret.length == n
        raise NotImplementedError, "Unexpected partial read from random device: only #{ret.length} for #{n} bytes"
      end
      return ret
    }
  rescue Errno::ENOENT
  end

  raise NotImplementedError, "No random device"
end
```

文件打开的 flags 是多种方式的 "与" 结果, 其中:

1. File::RDONLY 代表只读

2. File::NONBLOCK 代表打开或读写文件时非阻塞

3. File::NOCTTY 代表不会将打开的 IO 作为控制终端 [not to make opened IO the controlling terminal device](http://ruby-doc.org/core-2.0.0/File/Constants.html), 不是很懂这个的意图是什么.

打开 /dev/urandom 之后判断其是否为字符设备, 最后读取 n 个字节流.

内核产生随机数的原理是它维护了一个熵池来收集设备的环境噪音, 比如两次硬件中断的间隔, 两次鼠标或键盘的点击间隔, 或者两次磁盘操作的间隔等等. 根据这些数据产生熵估值, 熵估值描述池中包含的随机数的个数, 所以值越大, 随机性越好.

除了 /dev/urandom 之外, 系统还提供另外一个字符设备 /dev/random, 区别是后者在读取时只能返回小于或等于熵估值的随机字节, 若熵池空了会被阻塞, 而前者不会.
