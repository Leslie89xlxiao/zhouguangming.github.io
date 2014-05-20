---
layout: post
title: "How rails autoload files(1)"
description: ""
category:
tags: [rails]
---

在 Rails 的开发中有一个很舒服的地方是，在你修改了 Model 或 Controller 甚至 Route 代码之后，效果马上就可以在下一次请求中显示出来, 这个特性很大程度的节省了调试的时间, 今天我就来看看 Rails 是怎么来实现这个特性的， 由于涉及到的东西比较多，所以可能要分几次来完成整个的描述, 不想再像之前写 session 那篇文章那样，一是写着累，二是别人看着累。

<!--break-->

首先，我猜想完成这个特性要分两个步骤:

1. 监控文件变化

2. 文件变化后的回调

今天就先说第一部分： Rails 是如何知道我们修改了代码的, 以 Rails 3.2.12 为例。

完成这个功能最重要的一个类是： ActiveSupport::FileUpdateChecker

> It accepts two parameters on initialization. The first is an array
  of files and the second is an optional hash of directories. The hash must
  have directories as keys and the value is an array of extensions to be
  watched under that directory.
> This method must also receive a block that will be called once a path changes.

{% highlight ruby %}
def initialize(files, dirs={}, &block)
  @files = files
  @glob  = compile_glob(dirs)
  @block = block
  @updated_at = nil
  @last_update_at = updated_at
end

def updated_at #:nodoc:
  @updated_at || begin
    all = []
    all.concat @files.select { |f| File.exists?(f) }
    all.concat Dir[@glob] if @glob
    all.map { |path| File.mtime(path) }.max || Time.at(0)
  end
end

def compile_glob(hash) #:nodoc:
  hash.freeze # Freeze so changes aren't accidently pushed
  return if hash.empty?

  globs = []
  hash.each do |key, value|
    globs << "#{key}/**/*#{compile_ext(value)}"
  end
  "{#{globs.join(",")}}"
end

def compile_ext(array) #:nodoc:
  array = Array.wrap(array)
  return if array.empty?
  ".{#{array.join(",")}}"
end
{% endhighlight %}

根据上面的解释和代码，便可以很清晰的可以知道这个类的用法，类似如下，假设我要监控 '/tmp/test.txt' 和 '/tmp/' 目录下面所有以 'rb' 为拓展名的文件，那么我可以这么用：

{% highlight ruby %}
require 'active_support/file_update_checker'

files = ['/tmp/test.txt'] # 监控的文件
dirs  = { '/tmp/' => [:rb] } # 监控的目录
block = proc { puts 'file update' } # 回调

checker = ActiveSupport::FileUpdateChecker.new(files, dirs, &block)
{% endhighlight %}

初始化过程中 compile_glob 方法把 dirs 改变成 "/tmp/**/*.{rb}", 然后 updated_at 方法取到所有需要监控文件的数组，包括 files 和 dirs 相应的文件，再取出这些文件中最后一个更新过的文件的更新时间, 将这个时间分别复制给 @updated_at 和 @last_update_at, 这样初始化就结束了。

checker 提供三个接口出来：

{% highlight ruby %}
# Check if any of the entries were updated. If so, the updated_at
# value is cached until the block is executed via +execute+ or +execute_if_updated+
def updated?
  current_updated_at = updated_at
  if @last_update_at < current_updated_at
    @updated_at = updated_at
    true
  else
    false
  end
end

# Executes the given block and updates the counter to latest timestamp.
def execute
  @last_update_at = updated_at
  @block.call
ensure
  @updated_at = nil
end

# Execute the block given if updated.
def execute_if_updated
  if updated?
    execute
    true
  else
    false
  end
end
{% endhighlight %}

1. updated? 方法比较文件最后一次更新时间与 @last_update_at 比较, 如果有文件更改，把最新的更改时间赋值给 @updated_at。
2. execute 方法把最后一次更新时间赋值给 @last_update_at，并且执行 block 的内容。
3. execute_if_updated 方法就是把前面两个方法合并, 通常我们也是直接使用这个方法。

现在我们大概知道 Rails 是通过什么方式检测到文件发生变化的了，其实原理很简单，就是检测文件最后一次更新时间，若文件被修改了，则执行相应的代码，接下来还有一个问题，就是执行什么代码，使得我们的在应用中的更改很快就发生作用了呢，如果你比较了解 ruby，应该能猜到，大概就是重新 load 一下那些文件, 具体内容下次再说吧。
