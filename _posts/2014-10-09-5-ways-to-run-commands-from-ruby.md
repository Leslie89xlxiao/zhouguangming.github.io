---
layout: post
title: 5 ways to run commands from Ruby
---

#### Kernel#exec

执行子进程并退出当前 Ruby 进程.

```ruby
$ irb
  >> exec 'echo hello, `whoami`'
hello, zgm
```

#### Kernel#system

返回 `nil`, `true` 或 `false`, 子进程结束状态保存在 `$?` 中.

```ruby
$ irb
  >> system 'echo hello, `whoami`'
hello, zgm
  => true
  >> $?
  => #<Process::Status: pid 5930 exit 0>
```

#### Kernel#`

将子进程的标准输出作为函数返回值, 其结束状态保存在 `$?` 中.

```ruby
$ irb
  >> `echo hello, \`whoami\``
  => "hello, zgm\n"
  >> $?
  => #<Process::Status: pid 5931 exit 0>
```

#### IO#popen

返回一个连接子进程标准输入/输出的 `IO` 对象, 其结束状态不会保存在 `$?` 中.

```ruby
$ irb
  >> io = IO.popen("echo hello, `whoami`")
  => #<IO:fd 10>
  >> io.gets
  => "hello, zgm\n"
```

#### Open3#popen3

返回一个包括子进程标准输入/输出/错误以及一个等待子进程的线程的数组, 可以通过该线程 `value` 方法查看子进程的退出状态.

```ruby
$ irb
  >> stdin, stdout, stderr, thread = IO.popen("echo hello, `whoami`")
  => [#<IO:fd 10>, #<IO:fd 11>, #<IO:fd 13>, #<Thread:0x0000000189c2a8 run>]
  >> stdout.gets
  => "hello, zgm\n"
  >> thread.value
  => #<Process::Status: pid 6654 exit 0>
```
