---
layout: post
title: "A look at rails cookie-base session"
description: ""
category: 
tags: [rails, rack, http]
---
下文以 Rails 3.2.12 的源码为例解释 Rails cookie-base session 的工作原理

Rails 跟目录下面的 config/initializers/session_store.rb

<!--break-->

{% highlight ruby %}
Mytest::Application.config.session_store :cookie_store, key: '_mytest_session'
{% endhighlight %}

因此在键为 _mytest_session 的 Cookie 的值就是用来存储 session 的值的。

使用 crul 向服务器发送一个简单的 GET 请求，如下：

{% highlight bash %}
~/Workspace/mytest $: curl -i 127.0.0.1:3000

HTTP/1.1 200 OK 
Content-Type: text/html; charset=utf-8
X-Ua-Compatible: IE=Edge
Etag: "bea8252ff4e80f41719ea13cdf007273"
Cache-Control: max-age=0, private, must-revalidate
X-Request-Id: 1fc0e7bc7b5edf4bde55bbf68668e1e8
X-Runtime: 0.008572
Date: Wed, 27 Feb 2013 06:32:50 GMT
X-Rack-Cache: miss
Server: WEBrick/1.3.1 (Ruby/1.9.3/2013-02-22)
Content-Length: 14
Connection: Keep-Alive

Hello, World!
{% endhighlight %}

下面是 posts_controller.rb 下的 index action：

{% highlight ruby %}
def index
  render text: "Hello, World\n"
end
{% endhighlight %}

这时候并没有任何关于 Cookie 的 Header， 我们知道服务器通过设置健为 Set-Cookie 的 Header 来告诉浏览器设置 cookie 为什么值，但上一个请求并没有返回类似的 Header。
我稍微修改一下 index action 的代码:

{% highlight ruby %}
def index
  session[:foo] = :bar
  render text: "Hello, World\n"
end
{% endhighlight %}

再通过 curl 来请求，这次返回结果如下：

{% highlight bash %}
HTTP/1.1 200 OK 
Content-Type: text/html; charset=utf-8
X-Ua-Compatible: IE=Edge
Etag: "bea8252ff4e80f41719ea13cdf007273"
Cache-Control: max-age=0, private, must-revalidate
X-Request-Id: e528cdd41cd8e862fae2c584ea801d70
X-Runtime: 0.078162
Date: Wed, 27 Feb 2013 06:39:17 GMT
X-Rack-Cache: miss
Server: WEBrick/1.3.1 (Ruby/1.9.3/2013-02-22)
Content-Length: 14
Connection: Keep-Alive
Set-Cookie: _mytest_session=BAh7B0kiD3Nlc3Npb25faWQGOgZFRkkiJTY0ZTgwOTIyOTBkNDgxYmNlOTA4YWMyNTZjMzZiYmZhBjsAVEkiCGZvbwY7AEY6CGJhcg%3D%3D--6d40ea36c356e46b45794c1b9ee82607e19d1bf7; path=/; expires=Wed, 27-Feb-2013 06:39:27 GMT; HttpOnly

Hello, World!
{% endhighlight %}

在 Response 中找到了 Set-Cookie 的 Header。
通过 `rake middleware` 的结果猜测和 cookie 和 session 有关的 middleware 大概有两个：

`use ActionDispatch::Cookies`

`use ActionDispatch::Session::CookieStore`

看看它们都做了啥。

{% highlight ruby %}
# actionpack-3.2.12/lib/action-dispatch/middleware/cookies.rb line 335

class Cookies

  ...

  def initialize(app)
    @app = app
  end

  def call(env)
    cookie_jar = nil
    status, headers, body = @app.call(env)
  
    if cookie_jar = env['action_dispatch.cookies']
      cookie_jar.write(headers)
        if headers[HTTP_HEADER].respond_to?(:join)
          headers[HTTP_HEADER] = headers[HTTP_HEADER].join("\n")
        end
      end
  
      [status, headers, body]
    end
  end

  ...

end
{% endhighlight %}

这个 middleware 看上去貌似什么还没来得及做请求就到了下一个 middleware 中，也就是 `ActionDispatch::Session::CookieStore` ，下面是他的 initialize 和 call 方法：

{% highlight ruby %}
# actionpack-3.2.12/lib/action-dispatch/middleware/session/abstract_store.rb line 25

module Compatibility
  
  ...

  def initialize(app, options = {})
    options[:key] ||= '_session_id'
    # FIXME Rack's secret is not being used
    options[:secret] ||= SecureRandom.hex(30)
    super
  end

  ...

end

# rack-1.4.5/lib/rack/session/cookie.rb line 83

class Cookie < Abstract::ID

  ...

  def initialize(app, options={})
    @secrets = options.values_at(:secret, :old_secret).compact
    warn <<-MSG unless @secrets.size >= 1
    SECURITY WARNING: No secret option provided to Rack::Session::Cookie.
    This poses a security threat. It is strongly recommended that you
    provide a secret to prevent exploits that may be possible from crafted
    cookies. This will not be supported in future versions of Rack, and
    future versions will even invalidate your existing user cookies.
  
    Called from: #{caller[0]}.
    MSG
    @coder  = options[:coder] ||= Base64::Marshal.new
    super(app, options.merge!(:cookie_only => true))
  end

  ...

end

# rack-1.4.5/lib/rack/session/abstract/id.rb line 202

module Abstract

  ...

  class ID

    ...

    def initialize(app, options={})
      @app = app
      @default_options = self.class::DEFAULT_OPTIONS.merge(options)
      @key = @default_options.delete(:key)
      @cookie_only = @default_options.delete(:cookie_only)
      initialize_sid
    end

    ...

  end

  ...

end
{% endhighlight %}
{% highlight ruby %}
# rack-1.4.5/lib/rack/session/abstract/id.rb line 202

module Abstract

  ...

  class ID

    ...

    def call(env)
      context(env)
    end

    ...

  end

  ...

end
{% endhighlight %}

看上去真是相当凶残，慢慢分析一下。

1. 第一个 initialize 很简单，设置一些默认 key 和 secret options，在上面的设置中我们已经设置了 key，但是没有设置 secret 参数， 所以经过这一步生成一个 30 位的 secret。

2. 第二个 initialize 也比较简单，创建一个 @secrets 数组实例变量， 包括 secret 和 old_secret 参数，但是我们这里没有 old_secret 这个参数，所以下面的警告跳过，接下来创建一个 @coder 实例变量, 这个实例变量是 Base64::Marshal 的实例，这个类在该文件 48 行左右的位置定义的，作用是将 session cookies 以 Marshaled Base64 的形式存储。由于 rails 使用了自己的验证机制，所以上面的配置实际上没有用到。最后一行代码设置 cookie_only = true。

3. 第三个 initializer 的作用是首先将传入的 options 与 default options merge 一下，其中很多参数我们都没设置，就使用了默认的值，具体每个值什么含义就不一一解释了。下面将 key 和 cookie_only 复制给实例变量，然后把它们从 options 中删除。initialize_sid 这个方法把 sidbits 和 secure_random 从 options 中删除。

call 方法就一个 context 方法，代码如下：

{% highlight ruby %}
def context(env, app=@app)
  prepare_session(env)
  status, headers, body = app.call(env)
  commit_session(env, status, headers, body)
end
{% endhighlight %}

prepare_session 在 239 行， 作用是初始化 env['rack.session'] 和 env['rack.session.options'] 它们分别是 SessionHash 和 OptionsHash 的实例，这其中还有一个 session_was 的东西，它的作用是如果 env['rack.session'] 不为空， 则将它与新生成的实例 merge，通常它都为 nil 的，具体什么时候可能会产生，我没做深入研究。prepare_session 完成之后，就到了下一个 middleware 中，回来之后执行 commit_session 方法：

{% highlight ruby %}

# Acquires the session from the environment and the session id from
# the session options and passes them to #set_session. If successful
# and the :defer option is not true, a cookie will be added to the
# response with the session's id.

def commit_session(env, status, headers, body)
  session = env['rack.session']
  options = env['rack.session.options']

  if options[:drop] || options[:renew]
    session_id = destroy_session(env, options[:id] || generate_sid, options)
    return [status, headers, body] unless session_id
  end

  return [status, headers, body] unless commit_session?(env, session, options)

  session.send(:load!) unless loaded_session?(session)
  session = session.to_hash
  session_id ||= options[:id] || generate_sid

  if not data = set_session(env, session_id, session, options)
    env["rack.errors"].puts("Warning! #{self.class.name} failed to save session. Content dropped.")
  elsif options[:defer] and not options[:renew]
    env["rack.errors"].puts("Defering cookie for #{session_id}") if $VERBOSE
  else
    cookie = Hash.new
    cookie[:value] = data
    cookie[:expires] = Time.now + options[:expire_after] if options[:expire_after]
    set_cookie(env, headers, cookie.merge!(options))
  end

  [status, headers, body]
end
{% endhighlight %}

这段代码感觉很乱，理一理：

* 第一个 return 条件不成立， 因为 options[:drop] 和 options[:renew] 都使用默认值，也就是 false .

* 第二个 return 条件比较复杂，commit_session? 相关的代码如下：

{% highlight ruby %}
def commit_session?(env, session, options)
  if options[:skip]
    false
  else
    has_session = loaded_session?(session) || forced_session_update?(session, options)
    has_session && security_matches?(env, options)
  end
end

def loaded_session?(session)
  !session.is_a?(SessionHash) || session.loaded?
end

def forced_session_update?(session, options)
  force_options?(options) && session && !session.empty?
end

def force_options?(options)
  options.values_at(:renew, :drop, :defer, :expire_after).any?
end

def security_matches?(env, options)
  return true unless options[:secure]
  request = Rack::Request.new(env)
  request.ssl?
end
{% endhighlight %}

其中 !session.is_a?(SessionHash) 为 false，session.loaded? 的值呢？ 通过分析 SessionHash 的代码, 知道了当在我们 Rails App 中的 action 调用 session[:foo] = :bar 的时候，session 已经被 load 了，所以 session.loaded? 为 true， 从而 loaded_session? 为 true 从而 has_session 的值为 true ，由于 默认的 options[:secure] 为 false，所以 commit_session? 为 true， 条件不成立。

实际上 load session 的过程还是蛮复杂的，相关代码如下：

{% highlight ruby %}
# rack-1.4.5/lib/rack/session/abstract/id.rb line 50

class SessionHash < Hash

  ...

  def []=(key, value)
    load_for_write!
    super(key.to_s, value)
  end
  
  def load_for_write!
    load! unless loaded?
  end
  
  def load!
    id, session = @by.send(:load_session, @env)
    @env[ENV_SESSION_OPTIONS_KEY][:id] = id
    replace(stringify_keys(session))
    @loaded = true
  end

  ...

end

# rack-1.4.5/lib/rack/session/cookie.rb line 100

def load_session(env)
  data = unpacked_cookie_data(env)
  data = persistent_session_id!(data)
  [data["session_id"], data]
end

# rack-1.4.5/lib/rack/session/cookie.rb line 131

def persistent_session_id!(data, sid=nil)
  data ||= {}
  data["session_id"] ||= sid || generate_sid
  data
end

# actionpack-3.2.12/lib/action-dispatch/middleware/session/cookie-store.rb line 49

def unpacked_cookie_data(env)
  env["action_dispatch.request.unsigned_session_cookie"] ||= begin
    stale_session_check! do
      request = ActionDispatch::Request.new(env)
      if data = request.cookie_jar.signed[@key]
        data.stringify_keys!
      end
      data || {}
    end
  end
end

# actionpack-3.2.12/lib/action-dispatch/middleware/session/abstract_store.rb line 33

def generate_sid
  sid = SecureRandom.hex(16)
  sid.encode!('UTF-8') if sid.respond_to?(:encode!)
  sid
end


{% endhighlight %}

action 中的 session[:foo] = bar 触发了 []= 方法，最后的 load! 方法, 然后顺理成章的执行 load_session 方法，这个方法是关键，他首先调用 unpacked_cookie_data 方法, 这个方法又带我们回到了 ActionDispatch::Cookies 中, 相关代码如下：

{% highlight ruby %}
module ActionDispatch
  class Request
    def cookie_jar
      env['action_dispatch.cookies'] ||= Cookies::CookieJar.build(self)
    end
  end

  ...

  class Cookies 

    ...

    class CookieJar

      ...

      def self.build(request)
        secret = request.env[TOKEN_KEY]
        host = request.host
        secure = request.ssl?

        new(secret, host, secure).tap do |hash|
          hash.update(request.cookies)
        end
      enddef [](name)
        @cookies[name.to_s]
      end


      def initialize(secret = nil, host = nil, secure = false)
        @secret = secret
        @set_cookies = {}
        @delete_cookies = {}
        @host = host
        @secure = secure
        @closed = false
        @cookies = {}
      end

      def update(other_hash)
        @cookies.update other_hash.stringify_keys
        self
      end

      ...

    end

  ...

end
{% endhighlight %}

secret 为在 Rails 根目录下 config/initializers/secret_token.rb 定义的 secret_token, 类似如下代码：

{% highlight ruby %}
Mytest::Application.config.secret_token = '9e5998c078ca961e793d405b0aa5a3d5f92c3082ba805f565fd025c1c9c2c4793baad462bbb2e797bfcc16e2454b372c96ab30cd6d4d3c694c7c74ea9f38dd42'
{% endhighlight %}

host 是主机名或 IP 地址，在本例中为 '127.0.0.1'，secure 为请求是否为 ssl?，在本例中为 false，request.cookies 为 HTTP Request 中的 Cookie, 例如在 HTTP 的 Header 中加入: 

{% highlight bash %}
Cookie: 'foo:bar;baz:qux'
{% endhighlight %}

那么 request.cookies 的值为

{% highlight ruby %}
{"foo"=>"bar", "baz"=>"qux"}
{% endhighlight %}

在本例中并未传入任何 Cookie，所以齐值为空的 Hash 结构，即 {}。所以现在 Request.cookie 也就是 env['action_dispatch.cookies'] 的值应该很清晰了，Cookie::CookieJar 的一个实例，除了 host 和 secret 被更改外， 其余均为默认的值。接下来调用 signed 方法，下面是相关代码：

{% highlight ruby %}
def signed
  @signed ||= SignedCookieJar.new(self, @secret)
end

...

class SignedCookieJar < CookieJar
  def initialize(parent_jar, secret)
    ensure_secret_secure(secret)
    @parent_jar = parent_jar
    @verifier   = ActiveSupport::MessageVerifier.new(secret)
  end

  def [](name)
    if signed_message = @parent_jar[name]
      @verifier.verify(signed_message)
    end
  rescue ActiveSupport::MessageVerifier::InvalidSignature
    nil
  end
end
{% endhighlight %}

由上可知 signed 方法实际上是创建一个 @signed 的实例变量，这个实例变量是下面定义的 SignedCookieJar 的一个对象，该对象初始化的时候首先调用 ensure_secret_secure 方法，这个方法确保 @secret 一定要不为空，并且长度超过 30 位，@parent_jar 为上述的 CookieJar 对象，@verifier 对象为 ActiveSupport::MessageVerifier 的对象, [] 方法将上述的 参数 @key 传入的 @parent_jar，也就是那个 CookieJar 的对象当中，他的 [] 方法为：

{% highlight ruby %}
def [](name)
  @cookies[name.to_s]
end
{% endhighlight %}

我们已经知道了 @cookies 是一个空的 Hash 结构, 所有 @parent_jar 不会有结果, 返回 nil。回到上面 unpacked_cookie_data 的结果也是一个空的 Hash。 从而 persistent_session_id! 结果是一个只有一个 key： session_id 的 Hash 结构，它的值是被 generate_sid 临时创建出来的。回到上面用 load_session 方法，返回了一个数组，第一个元素是 session_id 的值，第二个元素是整个 Hash。回到 load! 方法，第一行得到了 id 和 session，随后将 id 的值给 env['rack.session.options'][:id]
，这会出发 OptionsHash#[] 方法，代码如下：

{% highlight ruby %}
class OptionsHash < Hash
  def initialize(by, env, default_options)
    @by = by
    @env = env
    @session_id_loaded = false
    merge!(default_options)
  end

  def [](key)
    load_session_id! if key == :id && session_id_not_loaded?
    super
  end

  private

  def session_id_not_loaded?
    !(@session_id_loaded || key?(:id))
  end

  def load_session_id!
    self[:id] = @by.send(:extract_session_id, @env)
    @session_id_loaded = true
  end
end
{% endhighlight %}

key == :id 为 true，@session_id_loaded 初始值为 false，key?(:id) 为 false，所以session_id_not_loaded? 为 true，执行 load_session_id! 方法。该方法向 @by 发送 extract_session_id 的消息，代码如下：

{% highlight ruby %}
def extract_session_id(env)
  unpacked_cookie_data(env)["session_id"]
end
{% endhighlight %}

又绕回来了，还记得之前执行这段代码是因为我们调用了 session[:foo] = :bar, 若没有调用的话，cookie 是在这里被 unpacked 的，具体的实现下面再说。再回到 load! 方法，接下来把自己的 key 都变成字符串类型，最后设置 @load 为 true。到此为止 load! 方法结束，回到 commit_session 方法，接下俩 session_id 等于之前通过 generate_sid 生成的 16 位字符串，到了 set_session 方法：

{% highlight ruby %}
# actionpack-3.2.12/lib/action-dispatch/middleware/session/cookie_store.rb line 61

def set_session(env, sid, session_data, options)
  session_data.merge!("session_id" => sid)
end
{% endhighlight %}

代码很简单，可能有些人会比较奇怪，刚才 options[:id] 明明是从 session_id 的值而来的，现在怎么又搞回去了？ 那是因为我们之前的流程只是其中的一种，我们仅仅使用了大多数的情况，可能实际情况要更加复杂，在这也不去细究了。因此 data 的值不为空， 第一个 if 跳过， options[:defer] = false, options[:renew] = false,
第二个 if 跳过，到了 else，创建临时变量 cookie， Hash 类型，设置 cookie value 和 expires，由于我们没有设置过，所以 expires 为空。将 cookie 和 options 合并，然后接下来调用 set_cookie 方法：

{% highlight ruby %}
# actionpack-3.2.12/lib/action-dispatch/middleware/session/cookie_store.rb line 65

def set_cookie(env, session_id, cookie)
  request = ActionDispatch::Request.new(env)
  request.cookie_jar.signed[@key] = cookie
end
{% endhighlight %}

是不是看上去有些眼熟，之前是 [] 方法，现在是 []= 方法:

{% highlight ruby %}
# actionpack-3.2.12/lib/action-dispatch/middleware/cookies.rb line 297

def []=(key, options)
  if options.is_a?(Hash)
    options.symbolize_keys!
    options[:value] = @verifier.generate(options[:value])
  else
    options = { :value => @verifier.generate(options) }
  end

  raise CookieOverflow if options[:value].size > MAX_COOKIE_SIZE
  @parent_jar[key] = options
end
{% endhighlight %}

进入第一个 if 语句，还记得 @verifier 方法吧, 它 ActiveSupport::MessageVerifier 的一个对象，generate 方法将返回一个字符串，这个字符串分为两部分，一部分是将 cookie[:value] 进行 Marshal dump 然后再 Base64 编码，另一部分是根据 @verifier 之前传入的实例变量 @serect 通过一定的签名算法得到的结果，两部分通过 '--' 连接。同理 verify 方法也是将加密后的字符串解密出来，若被篡改，则视为 session 无效。
然后调用 CookieJar#[]= 方法：

{% highlight ruby %}
def []=(key, options)
  if options.is_a?(Hash)
    options.symbolize_keys!
    value = options[:value]
  else
    value = options
    options = { :value => value }
  end

  handle_options(options)

  if @cookies[key.to_s] != value or options[:expires]
    @cookies[key.to_s] = value
    @set_cookies[key.to_s] = options
    @delete_cookies.delete(key.to_s)
  end

  value
end
{% endhighlight %}

handle_options 方法主要作用是处理 options 中一些参数的值，比如 path，domain 之类的，不去关心它了。接下来比较原始的 cookie-base session 和 新生成的是否相等，若不相等，设置为新的值，设置 @set_cookies 的 @delete_cookies 的值。

经过漫长的等待，总算跑完了 ActionDispatch::Session::CookieStore, 现在回到 ActionDispatch::Cookies, 执行 cookie_jar.write 方法：

{% highlight ruby %}
def write(headers)
  @set_cookies.each { |k, v| ::Rack::Utils.set_cookie_header!(headers, k, v) if write_cookie?(v) }
  @delete_cookies.each { |k, v| ::Rack::Utils.delete_cookie_header!(headers, k, v) }
end

def write_cookie?(cookie)
  @secure || !cookie[:secure] || always_write_cookie
end
{% endhighlight %}

这个方法其中一个重要的作用是把新的 cookie 的值写入 HTTP 的 Header 中，放在 Response 的 Set-Cookie 里面, 正如最初我们看得到那样。

Bingo. 整个过程总算结束了。
