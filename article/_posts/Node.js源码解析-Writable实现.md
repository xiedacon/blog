---
title: Node.js源码解析-Writable实现
date: 2017-08-06
tags:
---
# Node.js源码解析-Writable实现

对于一个 stream 模块来说，最基本的就是读和写。读由 Readable 负责，写则是由 Writable 负责。Readable 的实现上篇文章已经介绍过了，这篇介绍 Writable 的实现

## Base

在开始之前，先看看 Writable 的构造函数

```js
// lib/_stream_readable.js

function Writable(options) {
  // 只能在 Writable 或 Duplex 的上下文中执行
  if (!(realHasInstance.call(Writable, this)) &&
      !(this instanceof Stream.Duplex)) {
    return new Writable(options);
  }

  // Wrirable 的状态集
  this._writableState = new WritableState(options, this);

  // legacy
  this.writable = true;

  if (options) {
    if (typeof options.write === 'function')
      this._write = options.write;

    if (typeof options.writev === 'function')
      this._writev = options.writev;

    if (typeof options.destroy === 'function')
      this._destroy = options.destroy;

    if (typeof options.final === 'function')
      this._final = options.final;
  }

  Stream.call(this);
}

function WritableState(options, stream) {
  options = options || {};

  // object 模式标识
  this.objectMode = !!options.objectMode;

  if (stream instanceof Stream.Duplex)
    this.objectMode = this.objectMode || !!options.writableObjectMode;

  var hwm = options.highWaterMark;
  var defaultHwm = this.objectMode ? 16 : 16 * 1024;
  this.highWaterMark = (hwm || hwm === 0) ? hwm : defaultHwm;

  // highWaterMark 高水位标识
  // 此时，write() 返回 false
  // 默认 16k
  this.highWaterMark = Math.floor(this.highWaterMark);

  this.finalCalled = false;
  // drain 事件标识
  this.needDrain = false;
  // 刚调用 end() 时的状态标识
  this.ending = false;
  // end() 调用完成后的状态标识
  this.ended = false;
  this.finished = false;
  this.destroyed = false;

  var noDecode = options.decodeStrings === false;
  // 数据写入前，是否应该将 string 解析为 buffer
  this.decodeStrings = !noDecode;
  this.defaultEncoding = options.defaultEncoding || 'utf8';

  // 缓存中的数据长度
  // 不是真正的 buffer 长度，而是正在等待写入的数据长度
  this.length = 0;
  // writing 标识
  this.writing = false;
  // cork 标识
  this.corked = 0;
  // 标识是否有异步回调
  this.sync = true;
  // 标识正在写入缓存中的内容
  this.bufferProcessing = false;
  // _write() 和 _writev() 函数的回调
  this.onwrite = onwrite.bind(undefined, stream);
  // 调用 write(chunk, cb) 时的回调函数
  this.writecb = null;
  // 需写入的单个 chunk 块长度
  this.writelen = 0;

  // Writable 的缓冲池实现也是一个链表，其每个节点的结构如下:
  // {
  //   chunk,
  //   encoding,
  //   isBuf,
  //   callback,
  //   next
  // }  

  // 缓冲池头节点
  this.bufferedRequest = null;
  // 缓冲池尾节点
  this.lastBufferedRequest = null;
  // 缓冲池的大小
  this.bufferedRequestCount = 0;

  // 还需要执行的 callback 数量，必须在 finish 事件发生之前降为 0
  this.pendingcb = 0;
  this.prefinished = false;
  this.errorEmitted = false;

  // cork 的回调函数，最多只能有两个函数对象
  var corkReq = { next: null, entry: null, finish: undefined };
  corkReq.finish = onCorkedFinish.bind(undefined, corkReq, this);
  this.corkedRequestsFree = corkReq;
}
```

Writable 与 Readable 类似，也使用一个对象 ( WritableState ) 对状态和属性进行集中管理

在 Writable 的构造函数参数中，``options.write`` 函数是必须的，``options.writev`` 则是用于批量写入数据，可以选择实现

Writable 的缓冲池也是由链表实现，但与 Readable 不同的是，Writable 的缓冲池实现更简单，其节点结构如下:

  ```js
  {
    chunk,    // 数据块，可能是 object / string / buffer
    encoding, // 数据块编码
    isBuf,    // buffer 标识
    callback, // write(chunk, cb) 的回调函数
    next      // 下一个写入任务
  }
  ```

除此之外，WritableState 还使用 bufferedRequest、lastBufferedRequest、bufferedRequestCount 属性分别记录缓冲池的头、尾节点和节点数量

### 与 Duplex 的关系

在 Duplex 的源码中有这么一段注释

```js
// a duplex stream is just a stream that is both readable and writable.
// Since JS doesn't have multiple prototypal inheritance, this class
// prototypally inherits from Readable, and then parasitically from
// Writable.
```

意思是: Duplex 流既是 Readable 流又是 Writable 流，但是由于 JS 中的继承是基于原型的，没有多继承。所以 Duplex 是继承自 Readable，寄生自 Writable

寄生自 Writable 体现在两个方面：

  1. ``duplex instanceof Writable`` 为 true
  2. duplex 具有 Writable 的属性和方法

```js
// lib/_stream_writable.js

var realHasInstance;
if (typeof Symbol === 'function' && Symbol.hasInstance) {
  realHasInstance = Function.prototype[Symbol.hasInstance];
  Object.defineProperty(Writable, Symbol.hasInstance, {
    value: function(object) {
      if (realHasInstance.call(this, object))
        return true;

      return object && object._writableState instanceof WritableState;
    }
  });
} else {
  realHasInstance = function(object) {
    return object instanceof this;
  };
}

function Writable(options) {
  if (!(realHasInstance.call(Writable, this)) &&
      !(this instanceof Stream.Duplex)) {
    return new Writable(options);
  }
  // ...
}
```

可以看出，通过修改 Writable 的 ``Symbol.hasInstance`` 使得 ``duplex/writable instanceof Writable`` 为 ``true``。Writable 的构造函数也只能在 writable 或 duplex 的上下文中调用，使 duplex 具有 Writable 的属性

```js
// lib/_stream_duplex.js

util.inherits(Duplex, Readable);

var keys = Object.keys(Writable.prototype);
// 获取 Writable 的所有方法
for (var v = 0; v < keys.length; v++) {
  var method = keys[v];
  if (!Duplex.prototype[method])
    Duplex.prototype[method] = Writable.prototype[method];
}

function Duplex(options) {
  if (!(this instanceof Duplex))
    return new Duplex(options);

  Readable.call(this, options);
  Writable.call(this, options);
  // ...
}
```

遍历 Writable 原型上的方法，并添加到 Duplex 的原型上，使 duplex 具有 Writable 的方法

## Write 过程

Writable 的写过程比 Readable 的读过程简单得多，不用考虑异步操作，直接写入即可

```js
// lib/_stream_writable.js

Writable.prototype.write = function(chunk, encoding, cb) {
  var state = this._writableState;
  var ret = false;
  // 判断是否是 buffer
  var isBuf = Stream._isUint8Array(chunk) && !state.objectMode;
  // ...
  if (state.ended)
    // Writable 结束后继续写入数据会报错
    // 触发 error 事件
    writeAfterEnd(this, cb);
  else if (isBuf || validChunk(this, state, chunk, cb)) { 
    // 对 chunk 进行校验
    // 不能为 null
    // undefined 和非字符串只在 objectMode 下接受
    state.pendingcb++;
    ret = writeOrBuffer(this, state, isBuf, chunk, encoding, cb);
  }

  return ret;
};
```

``write()`` 函数对传入数据进行初步处理与校验后交由 ``writeOrBuffer()`` 函数继续处理

```js
// lib/_stream_writable.js

function writeOrBuffer(stream, state, isBuf, chunk, encoding, cb) {
  if (!isBuf) {
    // 非 buffer 的情况有: string 或 object
    // 为 object，代表是 objectMode，直接返回即可
    // 为 string，则需解码成 buffer
    var newChunk = decodeChunk(state, chunk, encoding);
    if (chunk !== newChunk) {
      isBuf = true;
      encoding = 'buffer';
      chunk = newChunk;
    }
  }
  var len = state.objectMode ? 1 : chunk.length;

  state.length += len;

  var ret = state.length < state.highWaterMark;
  if (!ret) state.needDrain = true;

  if (state.writing || state.corked) {
    // 正在写或处于 cork 状态
    // 将数据块添加到缓冲池链表为尾部
    var last = state.lastBufferedRequest;
    state.lastBufferedRequest = {
      chunk,
      encoding,
      isBuf,
      callback: cb,
      next: null
    };
    if (last) {
      last.next = state.lastBufferedRequest;
    } else {
      state.bufferedRequest = state.lastBufferedRequest;
    }
    state.bufferedRequestCount += 1;
  } else {
    // 写入数据
    doWrite(stream, state, false, len, chunk, encoding, cb);
  }

  return ret;
}
```

``writeOrBuffer()`` 函数对 chunk 进行处理后，根据 Writable 自身状态决定应何时写入数据

如果正在写入或处于 cork 状态，就将数据存储到缓冲池链表尾部，等待以后处理。否则，直接调用 ``doWrite()`` 写入数据

当缓存达到 highWaterMark 时，``writeOrBuffer()`` 返回 false，表示不应该再写入数据

```js
// lib/_stream_writable.js

function doWrite(stream, state, writev, len, chunk, encoding, cb) {
  state.writelen = len; // 写入的数据块长度
  state.writecb = cb; // 写入操作的回调函数
  state.writing = true;
  state.sync = true; // 同步状态标识
  if (writev) // 一次写入多个数据块
    stream._writev(chunk, state.onwrite);
  else // 一次写入单个数据块
    stream._write(chunk, encoding, state.onwrite);
  state.sync = false;
}
```

在 ``doWrite()`` 函数中，根据 writev 参数决定该执行 ``_write()`` 还是 ``_writev()`` 

``_write()`` 函数用于写入单个数据块，``_writev()`` 函数用于写入多个数据块

``_write()`` / ``_writev()`` 中的回调函数不是传入的 cb 而是 ``state.onwrite``，其定义如下:

  ```js
  this.onwrite = onwrite.bind(undefined, stream);
  ```

可知，写入完成后，执行 ``onwrite(stream, err)``

```js
// lib/_stream_writable.js

function onwrite(stream, er) {
  var state = stream._writableState;
  var sync = state.sync;
  var cb = state.writecb;

  // 更新 state
  onwriteStateUpdate(state);

  if (er) // 发生错误
    onwriteError(stream, state, sync, er, cb);
  else {
    var finished = needFinish(state);

    if (!finished &&
        !state.corked &&
        !state.bufferProcessing &&
        state.bufferedRequest) {
      // 清空缓冲池
      // 有 _writev() 函数时，执行 _writev() 一次写入多个数据块
      // 没有，则循环执行 _write() 写入单个数据块
      clearBuffer(stream, state);
    }

    if (sync) { // 代表写入操作是同步的，需要在 process.nextTick() 中执行 callback
      process.nextTick(afterWrite, stream, state, finished, cb);
    } else { // 代表写入操作是异步的，直接执行 callback 即可
      afterWrite(stream, state, finished, cb);
    }
  }
}
```

当写入过程中有错误发生时，会执行 ``onwriteError()``，继而调用 ``cb(err)`` 并触发 error 事件

如果写入过程正确执行，则先查看还有多少数据块正在等待写入，有多个，就执行 ``clearBuffer()`` 清空缓存，然后执行 ``afterWrite()``

```js
// lib/_stream_writable.js

function afterWrite(stream, state, finished, cb) {
  if (!finished)
    onwriteDrain(stream, state);
  state.pendingcb--;
  // 执行回调函数
  cb();
  // 检查是否应该结束 Writable
  finishMaybe(stream, state);
}
```

## Cork

当有大量小数据块需要写入时，如果一个个写入，会导致效率低下。Writable 提供了 ``cork()`` 和 ``uncork()`` 两个方法用于大量小数据块写入的情况

先将写操作柱塞住，等到缓存达到一定量后，再解除柱塞，然后一次性将存储的数据块写入，这个操作需要 ``_writev()`` 支持

```js
// lib/_stream_writable.js

Writable.prototype.cork = function() {
  var state = this._writableState;

  // 增加柱塞的次数
  state.corked++;
};
```

由于 ``cork()`` 函数可能会被多次调用，所以 ``state.corked`` 需要记录 ``cork()`` 调用的次数，是个 number

```js
// lib/_stream_writable.js

Writable.prototype.uncork = function() {
  var state = this._writableState;

  if (state.corked) {
    // 减少柱塞的次数
    state.corked--;

    if (!state.writing &&
        !state.corked &&
        !state.finished &&
        !state.bufferProcessing &&
        state.bufferedRequest)
      // 清空缓冲池
      clearBuffer(this, state);
  }
};
```

当 ``state.corked === 0`` 时，才能表示柱塞已经全部解除完毕，可以执行 ``clearBuffer()`` 来处理缓存中的数据

```js
// lib/_stream_writable.js

function clearBuffer(stream, state) {
  // 正在清空 buffer 的标识
  state.bufferProcessing = true;
  // 缓存的头节点
  var entry = state.bufferedRequest;

  if (stream._writev && entry && entry.next) {
    // _writev() 函数存在，且有一个以上数据块，就使用 _writev() 写入数据，效率更高
    var l = state.bufferedRequestCount;
    var buffer = new Array(l);
    var holder = state.corkedRequestsFree;
    holder.entry = entry;

    var count = 0;
    var allBuffers = true;
    // 取得所有数据块
    while (entry) {
      buffer[count] = entry;
      if (!entry.isBuf)
        allBuffers = false;
      entry = entry.next;
      count += 1;
    }
    buffer.allBuffers = allBuffers;

    // 写入数据
    doWrite(stream, state, true, state.length, buffer, '', holder.finish);

    state.pendingcb++;
    state.lastBufferedRequest = null;
    // 保证最多只有两个实例
    if (holder.next) { 
      state.corkedRequestsFree = holder.next;
      holder.next = null;
    } else {
      var corkReq = { next: null, entry: null, finish: undefined };
      corkReq.finish = onCorkedFinish.bind(undefined, corkReq, state);
      state.corkedRequestsFree = corkReq;
    }
  } else {
    // 一个个的写入
    while (entry) {
      var chunk = entry.chunk;
      var encoding = entry.encoding;
      var cb = entry.callback;
      var len = state.objectMode ? 1 : chunk.length;

      doWrite(stream, state, false, len, chunk, encoding, cb);
      entry = entry.next;
      // 如果写操作不是同步执行的，就意味着需要等待此次写入完成，再继续写入
      if (state.writing) {
        break;
      }
    }

    if (entry === null)
      state.lastBufferedRequest = null;
  }

  state.bufferedRequestCount = 0;
  // 修正缓存的头节点
  state.bufferedRequest = entry;
  state.bufferProcessing = false;
}
```

执行 ``clearBuffer()`` 时，根据是否有 ``_writev()`` 函数和待写入数据块数量，决定使用 ``_writev()`` 还是 ``_write()`` 写入数据

  * ``_writev()``: 会先将所有数据块包装成数组，然后写入。写入完成后，回调 ``corkReq.finish``
  * ``_write()``: 只需要将数据块一个个写入即可

在使用 ``_writev()`` 的情况下，写入完成后回调 ``corkReq.finish`` 也就是 ``onCorkedFinish()`` 函数

```js
// lib/_stream_writable.js

function onCorkedFinish(corkReq, state, err) {
  var entry = corkReq.entry;
  corkReq.entry = null;
  // 依次执行回调函数
  while (entry) {
    var cb = entry.callback;
    state.pendingcb--;
    cb(err);
    entry = entry.next;
  }
  // 保证最多只有两个实例
  if (state.corkedRequestsFree) {
    state.corkedRequestsFree.next = corkReq;
  } else {
    state.corkedRequestsFree = corkReq;
  }
}
```

根据缓冲池链表的顺序，依次执行写操作的回调函数

## End

每次调用 ``stream.write(chunk, cb)``，Writable 都会根据自身状态，决定是将 chunk 加到缓冲池，还是直接写入

当需要写入大量小数据块时，推荐先使用 ``cork()`` 将写操作柱塞住，待调用完毕后，再调用 ``uncork()`` 解除柱塞，然后一次性写入所有缓存数据

参考：
  * [https://github.com/nodejs/node/blob/master/lib/_stream_writable.js](https://github.com/nodejs/node/blob/master/lib/_stream_writable.js)
  * [https://github.com/nodejs/node/blob/master/lib/_stream_duplex.js](https://github.com/nodejs/node/blob/master/lib/_stream_duplex.js)