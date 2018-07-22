## 微服务与RPC

**背景：AWP系统迁移摒弃单体式开发`Monolithic`，全面拥抱微服务架构体系。**

在微服务中，使用什么协议来构建服务体系，一直是个热门话题。 争论的焦点集中在两个候选技术：（Binary）RPC or Restful。

---

## RPC（Remote Procedure Call）远程过程调用

---

![](https://obrxbqjbi.qnssl.com/blog/image/rpc-architecture.jpg)

---

**它的主要流程是Client -> Client Stub -> Network -> Server Stub -> Server 执行完成之后再进行返回。**

---

## RPC vs Restful

- 以Apache Thrift为代表的二进制RPC，支持多种语言（但不是所有语言），四层通讯协议，性能高，节省带宽。相对Restful协议，使用Thrift RPC，在同等硬件条件下，带宽使用率仅为前者的20%，性能却提升一个数量级。但是这种协议最大的问题在于，无法穿透防火墙。
- 以Spring Cloud为代表所支持的Restful 协议，优势在于能够穿透防火墙，使用方便，语言无关，基本上可以使用各种开发语言实现的系统，都可以接受Restful 的请求。 但性能和带宽占用上有劣势。

---

## RPC vs Restful

业内对微服务的实现，基本是确定一个组织边界，在该边界内，使用RPC; 边界外，使用Restful。这个边界，可以是业务、部门，甚至是全公司。

---

## 我们的现状

- 为了让模块间通信更加松散，可扩展，使用了Restful；
    - 网关以HTTP短连接方式分发微信请求到业务模块，qps单机不超过6k，且端口资源吃紧；
- 如果使用对内使用RPC，维持通信长连接池：
    - 一个简单的业务模块，为了RPC通信非要开发成RPCServer的形式，让人感到沮丧；
    - 网关需要动态维持与不同业务RPCServer的长连接池；
    - 模块间使用技术栈不同将增加RPC通信开发成本；

---


## 微服务之间通过RabbitMQ通信

RabbitMQ RPC

```php
$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo ' [.] Got ', $response, "\n";
```
---

## RabbitMQ RPC

> Callback Queue(回调队列)

**使用RabbitMQ实现RPC原理就是客户端发送请求消息，服务器接受并回复响应消息。为了能够接受到响应消息，在发送消息的时候需要提供一个回调队列。在发送消息时，往往使用reply_to属性来标识使用的回调队列。**

---

```php
list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);
$msg = new AMQPMessage(
    $payload,
    array('reply_to' => $queue_name)
);
$channel->basic_publish($msg, '', 'rpc_queue');
# ... then code to read a response message from the callback_queue ...
```

---

## correlation_id(关联标识)

之前的方法是给每个请求都创建一个回调队列，现实中请求数量往往较大，因此这种做法非常低效。另一个较好的方案是给每个客户端都只创建一个回调队列，使用`correlation_id`来标识收到的响应属于哪个请求，当我们从回调队列中接收到一个消息的时候，我们就可以查看这条属性从而将响应和请求匹配起来

---

![](https://images2015.cnblogs.com/blog/832799/201612/832799-20161224004437839-1074972304.png)

---

## rpc-client.php

[代码示例](https://github.com/jakubkulhan/bunny/blob/master/tutorial/6-rpc/rpc_client.php)

---

## rpc-server.php

[代码示例](https://github.com/jakubkulhan/bunny/blob/master/tutorial/6-rpc/rpc_server.php)

---

## 总结优势

- RabbitMQ中间件本身统一了通信协议，解决不同技术栈之间的隔阂，网关与业务模块只需专注于消息队列通信即可；

---
- 网关侧：
    - 长连接池只需关注消息队列即可；
    - 与业务模块的通信只需区分queue即可；
    - 请求不需要做鉴权，只有合法服务才能接入消息队列；

---

- 业务侧：
    - RPCServer的开发按照消费者开发标准即可（e.g.一个Lumen服务，只需开发一个Artisan 命令，而后交由Supervisor维持即可）；
    - 处理微信消息业务服务，全部自动变成内存常驻形；
    - 纵向扩展即可增加业务处理能力；
