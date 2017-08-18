---
title: Node.js源码解析-HTTP请求响应过程
date: 2017-08-18
tags: Node.js源码解析
---

# Node.js源码解析-HTTP请求响应过程

在 Node.js 中，起一个 HTTP Server 非常简单，只需要如下代码即可：

```js
const http = require('http')

http.createServer((req, res) => {
  res.end('Hello World\n')
}).listen(3000)
```

```
$ curl localhost:3000
Hello World
```

对，就这么简单。因为 Node.js 已经把具体实现细节给封装起来了，我们只需要调用 http 模块提供的方法即可

那么，一个请求是如何处理，然后响应的呢？让我们来看看源码

## Base

首先，让我们理一理思路

```
             _______
            |       | <== res
request ==> |   ?   | 
            |_______| ==> req
               /\
               ||
        http.createServer()
```

  * 先调用 ``http.createServer()`` 生成一个 ``http.Server`` 对象 ( 黑盒 ) 来处理请求
  * 每次收到请求，都先解析生成 req ( ``http.IncomingMessage`` ) 和 res ( ``http.ServerResponse`` )，然后交由用户函数处理
  * 用户函数调用 ``res.end()`` 来结束处理，响应请求

综上，我们的切入点有：
  * [http.createServer](#http.createServer)
  * [解析生成 req 和 res](#解析生成%20req%20和%20res)
  * [调用用户函数](#调用用户函数)
  * [res.end](res.end)

## http.createServer

让我们先来看看 ``http.createServer()``

```js
// lib/http.js

function createServer(requestListener) {
  return new Server(requestListener);
}

// lib/_http_server.js

function Server(requestListener) {
  if (!(this instanceof Server)) return new Server(requestListener);
  net.Server.call(this, { allowHalfOpen: true });

  if (requestListener) {
    this.on('request', requestListener);
  }

  this.on('connection', connectionListener);

  // ...
}
```

``http.createServer()`` 函数返回一个 ``http.Server`` 实例

该实例监听了 request 和 connection 两个事件
  * ``request 事件``：绑定 ``requestListener()`` 函数，req 和 res 准备好时触发
  * ``connection 事件``：绑定 ``connectionListener()`` 函数，连接时触发

用户函数是 ``requestListener()``，因此，我们需要知道 request 事件何时触发

```js
// lib/_http_server.js

function connectionListener(socket) {
  // ...

  // 从 parsers 中取一个 parser
  var parser = parsers.alloc();
  parser.reinitialize(HTTPParser.REQUEST);
  parser.socket = socket;
  socket.parser = parser;

  // ...

  state.onData = socketOnData.bind(undefined, this, socket, parser, state);

  // ...

  socket.on('data', state.onData);

  // ...
}

function socketOnData(server, socket, parser, state, d) {
  // ...
  var ret = parser.execute(d);
  // ...
}
```

当连接建立时，触发 connnection 事件，执行 ``connectionListener()``

这里的 socket 正是连接的 socket 对象，给 socket 绑定 data 事件用来处理数据，处理数据用到的 parser 是从 parsers 中取出来的

data 事件触发时，执行 ``socketOnData()``，最后调用 ``parser.execute()`` 来解析 HTTP 报文

值得一提的是 parsers 由一个叫做 FreeList ( [wiki](https://en.wikipedia.org/wiki/Free_list) ) 的数据结构实现，其主要目的是复用 parser

```js
// lib/internal/freelist.js

class FreeList {
  constructor(name, max, ctor) {
    this.name = name;
    this.ctor = ctor;
    this.max = max;
    this.list = [];
  }

  alloc() {
    return this.list.length ? this.list.pop() :
                              this.ctor.apply(this, arguments);
  }

  free(obj) {
    if (this.list.length < this.max) {
      this.list.push(obj);
      return true;
    }
    return false;
  }
}

module.exports = FreeList;
```

通过调用 ``parsers.alloc()`` 和 ``parsers.free(parser)`` 来获取释放 parser ( http 模块中 max 为 1000 )

## 解析生成 req 和 res

既然，HTTP 报文是由 parser 来解析的，那么，就让我们来看看 parser 是如何创建的吧

```js
// lib/_http_common.js

const binding = process.binding('http_parser');
const HTTPParser = binding.HTTPParser;

// ...

var parsers = new FreeList('parsers', 1000, function() {
  var parser = new HTTPParser(HTTPParser.REQUEST);

  parser._headers = [];
  parser._url = '';
  parser._consumed = false;

  parser.socket = null;
  parser.incoming = null;
  parser.outgoing = null;

  parser[kOnHeaders] = parserOnHeaders;
  parser[kOnHeadersComplete] = parserOnHeadersComplete;
  parser[kOnBody] = parserOnBody;
  parser[kOnMessageComplete] = parserOnMessageComplete;
  parser[kOnExecute] = null;

  return parser;
});

function parserOnHeadersComplete(versionMajor, versionMinor, headers, method, url, statusCode, statusMessage, upgrade, shouldKeepAlive) {
  // ...
    
  if (!upgrade) {
    skipBody = parser.onIncoming(parser.incoming, shouldKeepAlive);
  }

  // ...
}

// lib/_http_server.js

function connectionListener(socket) {
  // ...
  parser.onIncoming = parserOnIncoming.bind(undefined, this, socket, state);
  // ...
}
```

parser 是由 ``http_parser`` 这个库实现。不难看出，这里的 parser 也是基于事件的

在解析过程中，所经历的事件：
  * kOnHeaders：不断解析获取的请求头
  * kOnHeadersComplete：请求头解析完毕
  * kOnBody：不断解析获取的请求体
  * kOnMessageComplete：请求体解析完毕
  * kOnExecute：一次解析完毕 ( 无法一次性接收 HTTP 报文的情况 )

当请求头解析完毕时，对于非 upgrade 请求，可以直接执行 ``parser.onIncoming()``，进行响应

```js
// lib/_http_server.js

// 生成 response，并触发 request 事件
function parserOnIncoming(server, socket, state, req, keepAlive) {
  state.incoming.push(req);

  // ...

  var res = new ServerResponse(req);
  
  // ...

  if (socket._httpMessage) {
    state.outgoing.push(res);
  } else {
    res.assignSocket(socket);
  }

  // ...

  res.on('finish', resOnFinish.bind(undefined, req, res, socket, state, server));

  // ...

    server.emit('request', req, res);
  
  // ...
}

function resOnFinish(req, res, socket, state, server) {
  // ...

  state.incoming.shift();

  // ...

  res.detachSocket(socket);

  // ...
    var m = state.outgoing.shift();
    if (m) {
      m.assignSocket(socket);
    }
  }
}
```

可以看出，在源码中，对每一个 socket，维护了 ``state.incoming`` 和 ``state.outgoing`` 两个队列，分别用于存储 req 和 res

当 finish 事件触发时，将 req 和 res 从队列中移除

执行 ``parserOnIncoming()`` 时，最后会触发 request 事件，来调用用户函数

## 调用用户函数

就拿最开始的例子来说：

```js
const http = require('http')

http.createServer((req, res) => {
  res.end('Hello World\n')
}).listen(3000)
```

通过调用 ``res.end('Hello World\n')`` 来告诉 res，处理完成，可以响应并结束请求

## res.end

ServerResponse 继承自 OutgoingMessage，``res.end()`` 正是 ``OutgoingMessage.prototype.end()``

```js
// lib/_http_outgoing.js

OutgoingMessage.prototype.end = function end(chunk, encoding, callback) {
  // ...
  if (chunk) {
    // ...
    write_(this, chunk, encoding, null, true);
  } else if (!this._header) {
    this._contentLength = 0;
    // 生成响应头
    this._implicitHeader();
  }

  // ...
  // 触发 finish 事件
  var finish = onFinish.bind(undefined, this);

  var ret;
  if (this._hasBody && this.chunkedEncoding) {
    // trailer 头相关
    ret = this._send('0\r\n' + this._trailer + '\r\n', 'latin1', finish);
  } else {
    // 保证一定会触发 finish 事件
    ret = this._send('', 'latin1', finish);
  }

  this.finished = true;

  // ...
};

function write_(msg, chunk, encoding, callback, fromEnd) {
  // ...

  if (!msg._header) {
    msg._implicitHeader();
  }

  // ...

  var len, ret;
  if (msg.chunkedEncoding) { // 响应大文件时，分块传输
    if (typeof chunk === 'string')
      len = Buffer.byteLength(chunk, encoding);
    else
      len = chunk.length;

    msg._send(len.toString(16), 'latin1', null);
    msg._send(crlf_buf, null, null);
    msg._send(chunk, encoding, null);
    ret = msg._send(crlf_buf, null, callback);
  } else {
    ret = msg._send(chunk, encoding, callback);
  }

  // ...
}
```

执行函数时，如果给了 chunk，就先 ``write_(chunk)``，写入 chunk。再 ``_send('', 'latin1', finish)`` 绑定 finish 函数。待写入完成后，触发 finish 事件，结束响应

在写入 chunk 之前，必须确保 headers 已经生成，如果没有则调用 ``_implicitHeader()`` 隐式生成 headers

```js
// lib/_http_server.js

ServerResponse.prototype._implicitHeader = function _implicitHeader() {
  this.writeHead(this.statusCode);
};

ServerResponse.prototype.writeHead = writeHead;
function writeHead(statusCode, reason, obj) {
  // ...
  // 将 obj 和 this[outHeadersKey] 中的所有 header 放到 headers 中
  // ...

  var statusLine = 'HTTP/1.1 ' + statusCode + ' ' + this.statusMessage + CRLF;

  this._storeHeader(statusLine, headers);
}

// lib/_http_outgoing.js

OutgoingMessage.prototype._storeHeader = _storeHeader;
function _storeHeader(firstLine, headers) {
  // > GET /index.html HTTP/1.1\r\n
  // < HTTP/1.1 200 OK\r\n

  // ...
  // 校验 header 并生成 HTTP 报文头
  // ...

  this._header = state.header + CRLF;
  this._headerSent = false;

  // ...
}
```

headers 生成后，保存在 ``this._header`` 中，此时，响应报文头部已经完成，只需要在响应体之前写入 socket 即可

``write_()`` 函数，内部也是调用 ``_send()`` 函数写入数据

```js
// lib/_http_outgoing.js

OutgoingMessage.prototype._send = function _send(data, encoding, callback) {
  if (!this._headerSent) { // 将 headers 添加到 data 前面
    if (typeof data === 'string' &&
        (encoding === 'utf8' || encoding === 'latin1' || !encoding)) {
      data = this._header + data;
    } else {
      var header = this._header;
      if (this.output.length === 0) {
        this.output = [header];
        this.outputEncodings = ['latin1'];
        this.outputCallbacks = [null];
      } else {
        this.output.unshift(header);
        this.outputEncodings.unshift('latin1');
        this.outputCallbacks.unshift(null);
      }
      this.outputSize += header.length;
      this._onPendingData(header.length);
    }
    this._headerSent = true;
  }
  return this._writeRaw(data, encoding, callback);
};

OutgoingMessage.prototype._writeRaw = _writeRaw;
function _writeRaw(data, encoding, callback) {
  const conn = this.connection;

  // ...

  if (conn && conn._httpMessage === this && conn.writable && !conn.destroyed) {
    if (this.output.length) { // output 中有缓存，先写入缓存
      this._flushOutput(conn);
    } else if (!data.length) { // 没有则异步执行回调
      if (typeof callback === 'function') {
        nextTick(this.socket[async_id_symbol], callback);
      }
      return true;
    }
    // 直接写入 socket
    return conn.write(data, encoding, callback);
  }
  // 加入 output 缓存
  this.output.push(data);
  this.outputEncodings.push(encoding);
  this.outputCallbacks.push(callback);
  this.outputSize += data.length;
  this._onPendingData(data.length);
  return false;
}
```

对于 ``res.end()`` 来说，直接在 ``nextTick()`` 中触发 finish 事件，结束响应

## End

到这里，我们已经走完了一个请求从接收到响应的过程

除此之外，在 http 模块中，还考虑了许多的实现细节，非常值得一看

参考：
* [https://github.com/nodejs/node/blob/master/lib/http.js](https://github.com/nodejs/node/blob/master/lib/http.js)
* [https://github.com/nodejs/node/blob/master/lib/_http_server.js](https://github.com/nodejs/node/blob/master/lib/_http_server.js)
* [https://github.com/nodejs/node/blob/master/lib/_http_common.js](https://github.com/nodejs/node/blob/master/lib/_http_common.js)
* [https://github.com/nodejs/node/blob/master/lib/internal/freelist.js](https://github.com/nodejs/node/blob/master/lib/internal/freelist.js)
* [https://github.com/nodejs/node/blob/master/lib/_http_outgoing.js](https://github.com/nodejs/node/blob/master/lib/_http_outgoing.js)
