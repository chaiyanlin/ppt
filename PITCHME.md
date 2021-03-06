## 微服务与RPC

**背景：AWP系统迁移摒弃单体式开发`Monolithic`，全面拥抱微服务架构体系。**

在微服务中，使用什么协议来构建服务体系，一直是个热门话题。 争论的焦点集中在两个候选技术：（Binary）RPC or Restful。

---

## RPC（Remote Procedure Call）远程过程调用

**两台服务器A，B，一个应用部署在A服务器上，想要调用B服务器上应用提供的函数/方法，由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。**

---

![](https://obrxbqjbi.qnssl.com/blog/image/rpc-architecture.jpg)

---

**它的主要流程是Client -> Client Stub -> Network -> Server Stub -> Server 执行完成之后再进行返回。**

---

## RPC vs Restful

- 以Apache Thrift为代表的二进制RPC，支持多种语言（但不是所有语言），四层通讯协议，**性能高**，**节省带宽**。相对Restful协议，使用Thrift RPC，在同等硬件条件下，带宽使用率仅为前者的20%，性能却提升一个数量级。但是这种协议最大的问题在于，**无法穿透防火墙**。
- 以Spring Cloud为代表所支持的Restful 协议，优势在于**能够穿透防火墙**，**使用方便**，**语言无关**，基本上可以使用**各种开发语言实现**的系统，都可以接受Restful 的请求。 但**性能和带宽占用上有劣势**。

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

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1532368354101&di=97b49ee7a86bd10da68959bc0f61e992&imgtype=0&src=http%3A%2F%2Fimg.it610.com%2Fimage%2Finfo2%2Ffab754b4e27044e280efec17b7870eb3.jpg)

---

![](https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=183207246,1405132495&fm=27&gp=0.jpg)

---

消息属性(Message properties)

- 分发模式(deliveryMode): 标记一个消息是否需要持久化(persistent)或者是需要事务(transient)等
- 消息体类型(contentType): 描述消息中传递具体内容的编码方式，比如我们经常使用的JSON可以设置成:application/json
- 消息回应(replyTo): 回调队列
- 关联标识(correlationId): 用于将RPC的返回值关联到对应的请求

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

![](https://ss1.bdstatic.com/70cFvXSh_Q1YnxGkpoWK1HF6hhy/it/u=183207246,1405132495&fm=27&gp=0.jpg)

---

## rpc-client.php

[代码示例](https://github.com/jakubkulhan/bunny/blob/master/tutorial/6-rpc/rpc_client.php)

---

## rpc-server.php

[代码示例](https://github.com/jakubkulhan/bunny/blob/master/tutorial/6-rpc/rpc_server.php)

---

**归纳：**

- Server创建rpc_queue消息队列，并创建consumer处理rpc_queue的消息，也就是RPC请求的消息;
- Client发送RPC请求消息到rpc_queue;
- Server收到rpc_queue的消息，并执行并处理RPC，得到结果;
- Server将结果发送到Client的消息队列;
- Server发送ACK消息，标志一个RPC请求处理完成;
- Client通过`correlation_id`确定收到对应的应答后，发送ACK，标记接受到RPC的响应；

---

- 异常情况：
    - RPC Server处理RPC请求过程中异常退出，导致没有回复RPC结果，RPC Client也不知道，一直在等待结果

- 业务背景：
    - 微信服务`timeout`为5s；
    - 如果超时，不存在重试场景；

---

### 超时方案：

- 对队列中的消息设置`expiration`过期时间，过期没有收到`ACK`则在队列中销毁；
- 网关发送消息成功后，启动`协程`等待`RPC Response`，3s后放弃等待，返回微信服务空字符串（代表无消息返回）；

---

## 总结优势

- RabbitMQ中间件本身统一了通信协议，解决不同技术栈之间的隔阂，网关与业务模块只需专注与消息队列通信即可；

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
