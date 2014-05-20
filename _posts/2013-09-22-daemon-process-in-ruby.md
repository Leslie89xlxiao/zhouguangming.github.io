---
layout: post
title: "Daemon process in Ruby"
description: ""
category: 
tags: [ruby]
---

守护进程是一个在后台持续运行的进程, 独立于控制终端并且周期性的执行某种任务或者等待处理某些发生的事件. 守护进程的常见例子是 Web 服务器或数据库服务器等.

进程都是有编号的, 也就是进程的 pid. 进程都是衍生的, 也就是说, 绝大多数的进程都有一个父进程: ppid, 这里说绝大多数是因为有一个深奥的哲学问题: 鸡生蛋还是蛋生鸡的问题. 也就是第一个进程的父进程是谁呢?

<!--break-->

实际上, 当内核启动时会产生一个 init 的进程, 这个进程的 pid 是 1, ppid 是 0, 它是一切进程之父.( Mac 上不太相同, pid 为 1 的进程叫 launchd, 它的父进程叫 kernel_task )

下面的代码是 [rack](https://github.com/rack/rack/blob/master/lib/rack/server.rb#L320) 实现 daemon 方式, 我将一句一句的解释.

{% highlight ruby %}
# https://github.com/rack/rack/blob/master/lib/rack/server.rb#L320

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
{% endhighlight %}

首先, 在 1.9 以后 Ruby 出现了 Process.daemon 方法, 可以非常方便的将一个进程变成守护进程.

### Daemonizing a Process, Step by Step

`exit if fork`

这句代码巧妙的使用 fork 方法的特性: 在父进程中返回子进程 pid, 在子进程中返回 nil. 这样一来, 父进程退出, 子进程交给 init 进程来托管, 它的 ppid 变成了 1. 虽然它有一个父进程, 但这个父进程也是个继父, 或是干爹, 所以通常我们叫它孤儿进程.

`Process.setsid`

这段代码的作用有三个:

1. 该进程变成新会话首进程.

2. 该进程成为一个新的进程组的组长.

3. 该进程失去控制终端.

为了能够理解上面的事情, 我们需要对 进程, 进程组, 会话, 以及控制终端补补课.

### Process Groups and Session

进程的概念不必多说了, 是操作系统中正在运行的一个应用程序.

每一个进程都属于一个进程组, 每个进程组有一个首进程. 进程组是一个或多个进程的集合, 通常它们与一组作业相关联, 可以接受来自同一终端的各种信号. 每个进程组都有唯一的进程组 id. 

任何进程的子进程, 与父进程属于同一个组.

还记得之前提到的孤儿进程吗? 当一个进程 fork 了一个子进程, 然后这个进程在子进程退出前退出了, 我们叫这个子进程为孤儿进程, 这个很好理解. 但是如果子进程在父进程推出之前退出了, 并且父进程没有调用 [Process.wait](http://www.ruby-doc.org/core-2.0.0/Process.html#method-c-wait) 方法, 那么这个子进程会变成一个僵尸进程. 这是由于, 任何一个子进程, 在 exit 之后, 系统并不会立刻回收所有与之相关的资源, 而是保留一个关于这个进程的数据结构, 通过 Process.wait 来回收这个数据结构, 若不调用 Process.wait, 则这些数据结构无法被回收, 一直占着一个进程, 它已经没什么用了, 就好像僵尸一样, 只有一个躯壳.

接下来是会话, 一个会话是多个进程组的集合, 比如下面这个命令:

`git log | grep ruby | less`

每一个进程都是一个进程组(看上去这些组比较孤独, 但他们确实是一个组), 三个联合起来就成了一个会话.

一个会话可以有一个控制终端, 也可能没有控制终端(守护进程). 一个会话中的几个进程组可被分成一个前台进程组和几个后台进程组. 从控制终端发出的信号(Ctrl+C) 会被发给所有的前台进程组.

回到原来的例子中, Process.setsid 将当前的进程变成一个新的进程组以及会话的首进程, 这样一来, 通过控制终端的发给原来的进程组的信号就不会影响当前的进程了. 值得注意的是, 调用 Process.setsid 的进程必须不能是首进程, 所以有了第一个 fork 之后才调用这个方法.

现在当前进程是一个进程组以及会话的首进程, 虽然它已经脱离了控制终端, 但是从技术上来说, 还是可以给他分配一个的, 为了避免这种情况的发生, 出现了第二个 fork.

`exit if fork`

这个 fork 的作用是将当前进程变成一个不是首进程的进程, 因此系统不能分配终端给它. 现在好了, 进程变成了一个没有控制终端, 并且也无法分配控制终端的进程了.

`Dir.chdir '/'`

此行代码将工作目录更改为根目录, 确保了守护进程的当前工作目录不会因为这样那样的原因消失. 

{% highlight ruby %}
  STDIN.reopen "/dev/null"
  STDOUT.reopen "/dev/null", "a" 
  STDERR.reopen "/dev/null", "a" 
{% endhighlight %}

上面三行代码将进程的标准流重定向到 /dev/null 下.

以上就是通过 Ruby 实现一个守护进程的简单讲解, 短短几行代码看似简单, 实则藏着不少的奥秘啊.
