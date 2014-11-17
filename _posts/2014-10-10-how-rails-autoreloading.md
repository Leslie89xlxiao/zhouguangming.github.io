---
layout: post
title: How Rails Auto-Reloading
---

Rails 开发环境可以延迟加载类或模块, 还支持修改文件后重新加载, 前者提高了启动速度, 而后者方便了使用者调试.

#### 重新加载的工作原理

当请求到达 ActionDispatch::Reloader 时, 会使用 Rails Application 的 3 个 reloader(`YourApp::Application.reloaders`) 检查文件是否被修改, 他们分别监控的文件有:

  1. 用户自定义的类或者模块, 如 app/models/user.rb

  2. 路由文件, 如 config/routes.rb

  3. i18n 文件, 如 config/locales/en.yml

其中检查文件变化的类是 ActiveSupport::FileUpdateChecker, 其工作原理:

1. 修改文件会改变其 `File.mtime`, 这是此功能的前提

2. 找到被监控的文件中 mtime 最大的时间记为 current\_updated\_at

3. 将该时间与用于保存之前最后一次修改时间的实例变量 @last\_update\_at 进行比较, 若前者大于后者则认为有文件被修改过

4. 若发现有文件被修改过, 则将 current\_updated\_at 赋值给 @last\_update\_at 用于下次比较, 并执行各自预设的 reload 函数 --- 清空旧数据, 导入新数据

#### 自动加载的工作原理

程序第一次调用 User 类时, 大致流程如下:

  1. 由于还没有加载过定义该类的文件所以会抛出 NameError 异常, 但是 ActiveSupport::Dependencies 重新实现了 `Module#const_missing` 方法, 避免了直接抛出异常.

  2. 根据 **Convention Over Configuration** 原则, 在 ActiveSupport::Dependencies.autoload\_paths (如 app/models, app/controllers 等) 中遍历, 尝试找到对应的 user.rb 文件.

  3. 在找到对应的文件之后, 使用 `Kernel#load` 方法加载该文件, 否则报异常. 之所以使用 load 方法, 而不是常用的 require 方法, 是因为 require 只能加载一次, 而 load 可以重复加载.

当然 ActiveSupport::Dependencies 所做的远不止这些, 它处理了更多复杂的情况, 若希望知道里面的实现细节, 可以仔细看看[这个文件](https://github.com/rails/rails/blob/08754f12e65a9ec79633a605e986d0f1ffa4b251/activesupport/lib/active_support/dependencies.rb).

#### 更多

Rails 还提供一系列配置可以扩展自动加载的路径, 让其使用更加灵活:

```ruby
# 用户自定义类或文件 reloader
config.autoload_paths += %w(#{config.root}/lib)

# 路由文件 reloader
config.paths['config/routes.rb'] += Dir['config/routes/**/*.rb']

# i18n 文件 reloader
config.i18n.load_path += Dir[Rails.root.join('my', 'locales', '*.{rb,yml}').to_s]

# 关闭这个特性
config.cache_classes = false
```
