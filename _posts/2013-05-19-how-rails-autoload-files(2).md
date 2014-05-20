---
layout: post
title: "How rails autoload files(2)"
description: ""
category:
tags: [rails]
---

之前的一篇[博客](/2013/04/15/how-rails-autoload-files(1\)/) 介绍了 Rails 的开发环境是怎么知道用户更改了文件的，这篇博客介绍一下 Rails 是怎样动态的找到需要加载的文件的。

<!--break-->

Rails 的开发环境是不会一下子把所有要加载的类全部加载进来，而是在使用的时候动态加载，这样一方面是可以加快启动速度，另一方面可以实现更改文件后不需要重启 Rails 就生效。下面看看具体实现：

首先 ActiveRecord::Base 中 require 了 'active_record/dependencies', 是 Rails 实现这部分功能的主要代码。在 ActiveRecord::Dependencies::ModuleConstMissing 这个 module 重载了 Module 的 const_missing 方法:

{% highlight ruby %}
# activesupport-3.2.12/lib/active_support/dependencies.rb line 176

def const_missing(const_name, nesting = nil)
  klass_name = name.presence || "Object"

  unless nesting
    # We'll assume that the nesting of Foo::Bar is ["Foo::Bar", "Foo"]
    # even though it might not be, such as in the case of
    # class Foo::Bar; Baz; end
    nesting = []
    klass_name.to_s.scan(/::|$/) { nesting.unshift $` }
  end

  # If there are multiple levels of nesting to search under, the top
  # level is the one we want to report as the lookup fail.
  error = nil
  nesting.each do |namespace|
    begin
      return Dependencies.load_missing_constant Inflector.constantize(namespace), const_name
    rescue NoMethodError then raise
    rescue NameError => e
      error ||= e
    end
  end

  # Raise the first error for this set. If this const_missing came from an
  # earlier const_missing, this will result in the real error bubbling
  # all the way up
  raise error
end
{% endhighlight %}

当试图使用一个未定义的常量时会调用 const_missing 函数, 常量的搜寻路径可以参考[这篇文章](http://cirw.in/blog/constant-lookup)，当调用 const_missing 时， self 是常量的外部常量，比如使用 Foo::Bar 时，假设 Foo 常量不存在，在调用 const_missing 时的 self 是 Object，若 Foo 存在，但是 Foo::Bar 不存在，则调用 const_missing 时的 self 是 Foo 常量。

下面分析一个最简单的栗子：

在 Rails 中第一次调用 Post 这个 model 时(假设存在 /app/models/post.rb)，由于 post.rb 文件还没被加载，所以触发了 const_missing 方法，const_name 为 :Post，klass_name 为 "Object", nesting 为 ["Object"], 然后调用 Inflector.constantize(namespace) 返回名字为 namespace 的常量, 本例中为 Object，下面看看 Dependencies.load_missing_constant 方法:

{% highlight ruby %}
# activesupport-3.2.12/lib/active_support/dependencies.rb line 485

# Load the constant named +const_name+ which is missing from +from_mod+. If
# it is not possible to load the constant into from_mod, try its parent module
# using const_missing.
def load_missing_constant(from_mod, const_name)
  log_call from_mod, const_name


  unless qualified_const_defined?(from_mod.name) && Inflector.constantize(from_mod.name).equal?(from_mod)
    raise ArgumentError, "A copy of #{from_mod} has been removed from the module tree but is still active!"
  end

  raise NameError, "#{from_mod} is not missing constant #{const_name}!" if local_const_defined?(from_mod, const_name)

  qualified_name = qualified_name_for from_mod, const_name
  path_suffix = qualified_name.underscore

  file_path = search_for_file(path_suffix)

  if file_path && ! loaded.include?(File.expand_path(file_path).sub(/\.rb\z/, '')) # We found a matching file to load
    require_or_load file_path
    raise LoadError, "Expected #{file_path} to define #{qualified_name}" unless local_const_defined?(from_mod, const_name)
    return from_mod.const_get(const_name)
  elsif mod = autoload_module!(from_mod, const_name, qualified_name, path_suffix)
    return mod
  elsif (parent = from_mod.parent) && parent != from_mod &&
        ! from_mod.parents.any? { |p| local_const_defined?(p, const_name) }
    # If our parents do not have a constant named +const_name+ then we are free
    # to attempt to load upwards. If they do have such a constant, then this
    # const_missing must be due to from_mod::const_name, which should not
    # return constants from from_mod's parents.
    begin
      return parent.const_missing(const_name)
    rescue NameError => e
      raise unless e.missing_name? qualified_name_for(parent, const_name)
    end
  end

  raise NameError,
        "uninitialized constant #{qualified_name}",
        caller.reject {|l| l.starts_with? __FILE__ }
end
{% endhighlight %}

前两个异常不会抛出，qualified_name 是 from_mod 与 const_name 组成的常量字符串, 若 from_mod 为 Object 则省去, 结果为："Post"，path_suffix 将常量转换通过 underscore 方法将常量名转换为文件的 path 型，比如：

{% highlight ruby %}
"ActiveModel".underscore         # => "active_model"
"ActiveModel::Errors".underscore # => "active_model/errors"
{% endhighlight %}

search_for_file 方法在 Rails 的 autoload_paths 中寻找 'post.rb' 这个文件，找到 '/app/models/post.rb' 后，调用 require_or_load 方法加载这个文件，至于是 require 还是 load，之前也猜测了是 load，而实际上也却是是 load，这样才能重新加载修改过的文件, 但是 ActiveSupport::Dependencies 还是提供了一个接口用于 require 文件。

以上是 Rails 开发环境中一次最简单的常量查找过程, ActiveSupport::Dependencies 做了不止这些, 其实现也是 Rails 所提倡的 **习惯优于配置** 中 '习惯' 的一部分。
