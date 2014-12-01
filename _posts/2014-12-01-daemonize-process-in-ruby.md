---
layout: post
---

[rack](https://github.com/rack/rack/blob/master/lib/rack/server.rb#L320) 中实现守护进程方式如下:

```ruby
# https://github.com/rack/rack/blob/master/lib/rack/server.rb#L339

def daemonize_app
  if RUBY_VERSION < "1.9"
    exit if fork
    Process.setsid
    exit if fork
    Dir.chdir "/"
    STDIN.reopen "/dev/null"
    STDOUT.reopen "/dev/null", "a"
    STDERR.reopen "/dev/null", "a"
  else
    Process.daemon
  end
end
```

`exit if fork`

Fork 一个子进程, 父进程退出, 使子进程变成一个 [孤儿进程](http://zh.wikipedia.org/wiki/%E5%AD%A4%E5%84%BF%E8%BF%9B%E7%A8%8B), 从而可以调用下面的 `Process.setsid`.

`Process.setsid`

创建新的会话, 使该子进程成为新的会话首进程和新的进程组的组长并且脱离控制终端.

`exit if fork`

再次 Fork 一个孙进程, 子进程退出, 该步骤为了确保孙进程再也无法获取控制终端.

`Dir.chdir '/'`

将孙进程工作目录更改为根目录.

```ruby
STDIN.reopen '/dev/null'

STDOUT.reopen '/dev/null', 'a'

STDERR.reopen '/dev/null', 'a'
```

重定向标准流到 /dev/null
