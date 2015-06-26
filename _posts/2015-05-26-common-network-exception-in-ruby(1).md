---
layout: post
title: Common Network Exception in Ruby(1)

---

### Errno::EHOSTUNREACH

当试图连接一个无法通过 ARP 协议获取 MAC 地址的 IP 地址时抛出该异常.

```ruby
require 'socket'

begin
  # 假设 192.169.1.100 这个 IP 与请求方在同一子网, 但是该子网内不存在该主机.
  client = TCPSocket.new('192.168.1.100', 80)
rescue => error
  puts error.class
  puts error.message
end

=>

Errno::EHOSTUNREACH
No route to host - connect(2) for "192.168.1.100" port 3000
```

### Errno::ETIMEDOUT

当建立连接时, 请求方由于未收到接收方的 ACK 报文, 经过 N 次超时重传之后抛出该异常.

```ruby
require 'socket'

begin
  # 已通过 iptables DROP 掉所有到达 3000 端口的 TCP 数据包.
  client = TCPSocket.new('localhost', 3000)
rescue => error
  puts error.class
  puts error.message
end

=>

Errno::ETIMEDOUT
Connection refused - connect(2) for "localhost" port 3000
```

### Errno::ECONNREFUSED

当建立连接时, 接收方对应的端口未开启会传回一个 RST 报文抛出该异常.

```ruby
require 'socket'

begin
  # 接收方并未开启 3000 端口的服务.
  client = TCPSocket.new('localhost', 3000)
rescue => error
  puts error.class
  puts error.message
end

=>

Errno::ECONNREFUSED
Connection refused - connect(2) for "localhost" port 3000
```

### Errno::EPIPE

当接收方正常关闭连接并发送 FIN 报文后, 如果发送方执行写操作, 则接收方会返回一个 RST 报文, 如果发送方继续执行写操作, 则抛出该异常.

```ruby
# server.rb

require 'socket'

begin
  server = TCPServer.new('localhost', 3000)

  client = server.accept
  # 让客户端先关闭, 并发送 FIN 报文
  sleep 1
  # 此时客户端发送 RST 报文
  client.write 'hello'
  # 抛出 Error::EPIPE 异常
  client.write 'world'
rescue => error
  puts error.class
  puts error.message
end

=>

Errno::EPIPE
Broken pipe
```

```ruby
# client.rb

require 'socket'

begin
  client = TCPSocket.new('localhost', 3000)
  client.close
end
```

### Errno::ECONNRESET

当接收方异常关闭连接并发送 RST 报文后, 如果发送方继续执行读或者写操作, 则抛出该异常.

```ruby
# server.rb

require 'socket'

begin
  server = TCPServer.new('localhost', 3000)
  client = server.accept
  client.write 'hello'
  # 等待客户端异常退出, 发送 RST 报文
  sleep 2
  # 抛出 Errno::ECONNRESET
  client.write 'world'
rescue => error
  puts error.class
  puts error.message
end

=>

Errno::ECONNRESET
Connection reset by peer
```

```ruby
# client.rb

require 'socket'

begin
  client = TCPSocket.new('localhost', 3000)
  # 等待服务端发送数据过来
  sleep 1
  # 由于缓冲区仍有数据, 此时关闭 socket 会导致发送一个 RST 报文到服务端
  client.close
end

```
