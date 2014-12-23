---
layout: post
---

在 Ruby 中使用 [callcc](http://www.ruby-doc.org/core-2.1.5/Continuation.html) 实现 [continuation](http://en.wikipedia.org/wiki/Continuation):

```ruby
require 'continuation'

3.times do |i|
  puts i

  $cc = callcc { |cc| cc } if i == 1
end

$cc.call if $cc

# output

0
1
2
2
```

1. 当 i 等于 1 时 `callcc { |cc| cc }` 返回一个 Continuation 对象赋值给全局变量 $cc

2. 第一次跳出迭代之后, 执行 `$cc.call`, 程序回到迭代 i = 1 时的状态, `callcc { |cc| cc }` 返回 nil 赋值给 $cc, 继续迭代打印出 2

3. 第二次跳出迭代以后, 由于此时 $cc = nil 所以不会再次执行 `$cc.call`, 程序结束

下面是一个利用 callcc 实现的简易 [amb operator](http://community.schemewiki.org/?amb)

```ruby
require 'continuation'

$backtracks = []

def amb *choices
  if choices.empty?
    if $backtracks.empty?
      raise 'No backtrack'
    else
      $backtracks.pop.call
    end
  else
    callcc do |cc|
      $backtracks.push(cc)

      return choices.first
    end

    amb *choices[1..choices.length]
  end
end

# example

x = amb 0, 1, 2, 3, 4
y = amb 0, 1, 2, 3, 4

amb if x * y != 4
amb if x + y != 4

puts "x = #{x}, y = #{y}"

# output

x = 2, y = 2
```
