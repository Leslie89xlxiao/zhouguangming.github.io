---
layout: post
title: How Rails Auto-Reloading
---

Rails 开发环境可以延迟加载类或模块, 还支持修改文件后重新加载, 前者提高了启动速度, 而后者方便了使用者调试.

重新加载的工作原理如下:

当请求进入到 ActionDispatch::Reloader 时, 它会检查文件是否被修改, 若被修改, 则清除已经加载的内容. 文件检查使用 ActiveSupport::FileUpdateChecker 这个类, 原理是当文件被改动后其 mtime 属性也会随之变为最后一次保存时间, 利用这一特性便可以知道文件是否有更新. 若文件被更新了, 则 ActiveSupport::Dependencies 会使用 Module#remove_const 删除掉已经加载的类或模块.

自动加载的工作原理如下:

假如程序调用 User 类, 由于还没有加载过定义该类的文件所以会抛出 NameError 异常, 但是 ActiveSupport::Dependencies 重新实现了 Module#const\_missing 方法, 根据 **convention over configuration** 原则在 ActiveSupport::Dependencies.autoload\_paths(如 app/models, app/controllers 等) 中找到对应的 user.rb 文件, 并且使用 Kernel#load 方法加载该文件, 之所以使用 load 方法, 而不是常用的 require 方法, 是因为 require 只能加载一次, 而 load 可以重复加载, 从而实现重新加载的目的.
