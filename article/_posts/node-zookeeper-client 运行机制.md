---
title: node-zookeeper-client 运行机制
date: 2019-06-22
tags: zookeeper
---

ZooKeeper ( 简称 zk ) 是一个开源的分布式协调服务，其常用做分布式服务的注册中心

本文要讲的是 Node.js 的 zk sdk -- node-zookeeper-client

在这之前，先了解一下 zk 相关的基本知识

* sessionId：客户端连接 session，默认 30s 后过期
* xid：客户端发送消息序号，zk 服务端进行响应时，xid 相同
* zxid：``ZooKeeper Transaction Id``，用于保证 zk 分布式一致性
* Jute：zk 底层的序列化工具
* packet：由 header 和 payload 组成，序列化后 buffer 结构类似 ``[总长度, header, payload]``

# 使用示例

先从使用示例开始

```js
/**
 * xiedacon created at 2019-06-14 10:07:46
 *
 * Copyright (c) 2019 Souche.com, all rights reserved.
 */
'use strict';

const ZK = require('node-zookeeper-client');
const util = require('util');

const client = ZK.createClient('127.0.0.1:2181');

client.getChildren = util.promisify(client.getChildren);

(async () => {
  console.log('Result:', await client.getChildren('/xxx'));
})().catch((err) => {
  console.log(err);
});

client.connect();
```

上面的代码总共分成 3 步：

1. 调用 ``ZK.createClient('127.0.0.1:2181')``，创建 zk 客户端实例
2. 调用 ``client.connect()``，与 zk 服务端建立连接
3. 调用 ``client.getChildren('/xxx')``，获取 /xxx 节点下的所有子节点

## 1. 创建 zk 客户端

zk 客户端可大致分为三块：

* client：作为暴露 API 的一层 adapt，主要用于组装 request，对应文件 ``index.js``
* connectionManager：管理客户端与服务端的 socket 连接，以及相关事件，对应文件 ``lib/connectionManager.js``
* watcherManager：管理客户端的 watches，对应文件 ``lib/watcherManager.js``

## 2. 建立 zk 连接

1. 调用 ``client.connect()``，connect 方法内调用 ``connectionManager.connect()``，将 connectionManager 状态置为 **CONNECTING**
2. 通过 ``findNextServer()`` 获取一个 zk 服务器地址
3. 为 socket 绑定 ``connect``、``data``、``drain``、``close``、``error`` 事件
4. 如果 tcp 连接建立成功，则触发 connect 事件，并执行 ``onSocketConnected`` 方法
 * 清除连接超时定时器
 * 生成 ConnectRequest 并写入 socket
 * 如果存在 auth info 则生成 AuthPacket 并放入 packetQueue
 * 如果存在 watches 则生成 SetWatches 并放入 packetQueue
5. 如果 tcp 连接建立失败，则分别触发 ``error``、``close`` 事件，并执行 ``onSocketError``、``onSocketClosed`` 方法
 * 清除连接超时定时器
 * 清除 pendingQueue 内的等待包
 * 根据 connectionManager 状态决定是否重连
 * 将 connectionManager 状态置为 **DISCONNECTED**
 * 重连则重新执行 1，不重连则将 connectionManager 状态置为 **CLOSED**
6. 如果 zk 服务器允许 client 接入
 * 触发 data 事件并执行 ``onSocketData`` 方法，此时 connectionManager 状态为 **CONNECTING**
 * 构造 ConnectResponse 并解析 socket 数据
 * 如果 ``connectResponse.timeOut <= 0`` 意味着当前 client 的 session 过期：将 connectionManager 状态置为 **SESSION_EXPIRED**，稍后 zk 服务器将自动关闭 socket，即触发 5
 * 其他情况意味着连接建立成功：设置相关数据并将 connectionManager 状态置为 **CONNECTED**，主动触发 socket ``drain``，发送阻塞 request，使数据流动起来
7. 如果 zk 服务器拒绝 client 接入
 * zk 服务器不会返回任何信息，直接关闭 socket，即触发 5

PS：其实并没有触发 socket ``drain`` 的代码，而是调用的 ``onPacketQueueReadable`` 方法，但它们的效果一样

![](/images/node-zookeeper-client%20运行机制/1.png)

## 3. 发送消息

1. 调用 ``client.getChildren('/xxx')``
2. client 生成对应的 request 结构体，并调用 ``connectionManager.queue`` ，将 request 放入 packetQueue 中，同时触发 socket ``drain`` 
3. 在 ``onPacketQueueReadable`` 方法中，对于非 Auth、Ping 等的外部 request，设置 xid
4. 将 request 序列化并写入 socket，同时 request 放入 pendingQueue
5. 当 zk 服务端处理完成后，会返回处理结果，触发 ``data`` 事件，此时 connectionManager 状态为 **CONECTED**
6. 构造 ReplyHeader 并解析 socket 数据
7. 如果 xid 为内部 xid 则进行内部处理，否则对比 pendingQueue 队头 request 的 xid 与当前 xid 是否一致
 * 如果一致，则构造相应 Response 并执行 ``client.getChildren`` 的回调方法
 * 如果不一致，意味着客户端出现可不预知错误，直接关闭 socket 连接

PS：其实并没有触发 socket ``drain`` 的代码，而是触发 packetQueue 的 ``readable`` 事件，但它们的效果一样

# connectionManager 状态机

![](/images/node-zookeeper-client%20运行机制/2.png)

* CONNECTING：zk 连接建立中
* CONNECTED：zk 连接已建立完成
* DISCONNECTED：zk 连接已断开
* CLOSED：zk 客户端已关闭
* CLOSING：zk 客户端关闭中
* SESSION_EXPIRED：zk 客户端 session 过期，客户端无法处理，最终会导致客户端关闭
* AUTHENTICATION_FAILED：zk 客户端 auth 认证失败，客户端无法处理，最终会导致客户端关闭

各方法与状态机的关系如下：

![](/images/node-zookeeper-client%20运行机制/3.png)

# 存在缺陷

## session 过期无法重连

### 造成原因

node-zookeeper-client 本身提供断线重连的能力

对于长期断线 ( 30s 以上 )，连接重新建立后，当前客户端的 session 已经过期，connectionManager 将进入 **SESSION_EXPIRED** 状态，此时调用 ``queue`` 方法会抛出 ``SESSION_EXPIRED`` 错误

这一问题在 [zookeeper 文档](https://wiki.apache.org/hadoop/ZooKeeper/FAQ#A3) 中也有提到：

> It means that the client was partitioned off from the ZooKeeper service for more the the session timeout and ZooKeeper decided that the client died.

当客户端收到 ``SESSION_EXPIRED`` 错误时，那意味着当前客户端已经从 ZooKeeper 服务中断开，即 sesssion 过期，ZooKeeper 判定当前客户端应当关闭。

> If the client is only reading state from ZooKeeper, recovery means just reconnecting. In more complex applications, recovery means recreating ephemeral nodes, vying for leadership roles, and reconstructing published state.

如果客户端以只读方式连接的 ZooKeeper，那么只需要重新连接就好了。但在更复杂的情况下，重连就意味着重新创建临时节点，参与 leader 竞争，重建已发布的状态。

因此 node-zookeeper-client 的做法也合情合理，当 session 过期时直接禁止发送，由调用程序决定是否重连，重连后需要做什么操作

但在大多数情况下，node-zookeeper-client 只会作为一个简单的客户端连接 zookeeper 服务，不会参与 zk 选举，因此，它最多需要做的事情也只是恢复临时节点

### 解决方案

目前的方案是 session 过期自动重连，重连成功后将会进入 **CONNECTED** 状态并触发 ``connect`` 事件，在事件内重新注册临时节点

1. 发现 session 过期时，直接重置 sessionId，下次重连时将会重新获取 sessionId
2. socket 关闭时，将 pendingQueue 内的包，移到 packetQueue 队头，避免丢包
3. 内部包直接从 socket 发送不走 packetQueue，避免阻塞，如 AuthPacket、SetWatches、Ping

## zk 重启无法重连

### 造成原因

在 zk 集群中，不同服务节点之间通过 [ZAB 协议](https://zhuanlan.zhihu.com/p/27335748)，保证分布式一致性。zxid ( ZooKeeper Transaction Id ) 就是用于保证分布式一致性的标识符

zk 直接挂掉或者手动重启都将会导致集群 zxid 重置。当客户端重连时，由于客户端本身存储的 xzid 大于 zk 集群 xzid，此时，zk 集群将拒绝客户端连接。外部表现为 connectionManager 疯狂执行 connect 操作，所有调用阻塞，zk 日志输出 ``Refusing session request for client /192.168.0.1:33400 as it has seen zxid 0x300000012 our last zxid is 0x0 client must try another server``

虽然 zk 集群判断出 zxid 有误，但直接断开连接的处理方式确实是有待考量。因为通过这种方式处理，客户端完全不知道发生了什么，只能通过重连来解决，然而重连又连接不上，这就形成了一个死循环

### 解决方案

由于客户端无需在意 zk 集群版本，因此，可以在每次连接时，直接重置 zxid 即可

# 相关链接

* https://github.com/alexguan/node-zookeeper-client
* https://github.com/apache/zookeeper/tree/master
* https://github.com/apache/zookeeper/blob/master/zookeeper-jute/src/main/resources/zookeeper.jute
* https://www.cnblogs.com/leesf456/p/6091208.html
* https://wiki.apache.org/hadoop/ZooKeeper/FAQ
* https://zhuanlan.zhihu.com/p/27335748
* https://www.cnblogs.com/luxiaoxun/p/4887452.html
* https://issues.apache.org/jira/browse/ZOOKEEPER-832
