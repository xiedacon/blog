# Readable 实现

想要了解 Readable 的实现，最好的方法是顺着 Readable 的 Birth-Death 走一遍

## Base

在了解 Readable 的 Birth-Death 之前，先看看 Readable 的构造函数

```js
// lib/_stream_readable.js

function Readable(options) {
  if (!(this instanceof Readable))
    return new Readable(options);

  // Readable 流的状态集
  this._readableState = new ReadableState(options, this);

  // legacy
  this.readable = true;

  if (options) {
    if (typeof options.read === 'function')
      // 真实数据来源，Readable.prototyoe._read() 函数会抛出异常，因此必须有options.read
      this._read = options.read;

    if (typeof options.destroy === 'function')
      this._destroy = options.destroy;
  }

  Stream.call(this);
}

function ReadableState(options, stream) {
  options = options || {};

  // object 模式标识
  this.objectMode = !!options.objectMode;

  if (stream instanceof Stream.Duplex)
    this.objectMode = this.objectMode || !!options.readableObjectMode;

  var hwm = options.highWaterMark;
  var defaultHwm = this.objectMode ? 16 : 16 * 1024;
  this.highWaterMark = (hwm || hwm === 0) ? hwm : defaultHwm;

  // highWaterMark 高水位标识
  // 内部缓存高于 highWaterMark 时，会停止调用 _read() 获取数据
  // 默认 16k
  this.highWaterMark = Math.floor(this.highWaterMark);

  // Readable 流内部缓冲池，是一个 buffer 链表
  // 之所以不用数组实现，是因为链表增删头尾元素更快
  this.buffer = new BufferList();
  // 缓存大小
  this.length = 0;
  // pipe 的流
  this.pipes = null;
  this.pipesCount = 0;
  // flow 模式标识
  this.flowing = null;
  // Readable 状态标识，为 true 表示数据源已读取完毕
  // 此时 Readable 中可能还有数据，不能再向缓冲池中 push() 数据
  this.ended = false;
  // Readable 状态标识，为 true 表示 end 事件已触发
  // 此时 Readable 中数据已读取完毕，不能再向缓冲池中 push() 或 unshift()　数据
  this.endEmitted = false;
  // Readable 状态标识，为 true 表示正在调用 _read() 读取数据
  this.reading = false;
  this.sync = true;
  // 标识需要触发 readable 事件
  this.needReadable = false;
  // 标识已触发 readable 事件
  this.emittedReadable = false;
  this.readableListening = false;
  this.resumeScheduled = false;
  this.destroyed = false;
  this.defaultEncoding = options.defaultEncoding || 'utf8';
  this.awaitDrain = 0;
  this.readingMore = false;

  // 解码器
  this.decoder = null;
  this.encoding = null;
  if (options.encoding) {
    if (!StringDecoder)
      StringDecoder = require('string_decoder').StringDecoder;
    this.decoder = new StringDecoder(options.encoding);
    this.encoding = options.encoding;
  }
}
```

在 Readable 的构造函数中，可通过 options 传入参数，其中 ``options.read`` 函数是必需的

``readable._readableState`` 中保存了 Readable 的各种状态与属性

## Birth-Death

在这里将 Readable 的 Birth-Death 分为五个状态:

> 表中为 this._readableSate 的属性

* start: 初始状态，Readable 刚刚被创建，还未调用 ``readable.read()``

  |length|reading|ended|endEmitted|
  |--|--|--|--|
  |0|false|false|false|

* reading: 代表正在从数据源中读取数据，此时缓存大小 ``this._readableSate.length`` 小于 ``highWaterMark``，应读取数据使缓存达到 ``highWaterMark``

  |length|reading|ended|endEmitted|
  |--|--|--|--|
  |< highWaterMark|true|false|false|

* read: Readable 从数据源读取数据后的相对稳定状态

  |length|reading|ended|endEmitted|
  |--|--|--|--|
  |>= highWaterMark|false|false|false|

* ended: 数据已经全部读取完成（``push(null)``），此时 ``push(chunk)`` 会报 ``stream.push() after EOF`` 错误

  |length|reading|ended|endEmitted|
  |--|--|--|--|
  |>= 0|false|true|false|

* endEmitted: end 事件触发完成，此时 ``unshift(chunk)`` 会报 ``stream.unshift() after end event`` 错误

  |length|reading|ended|endEmitted|
  |--|--|--|--|
  |0|false|true|true|

它们之间的关系如下:

```
         1           4         5
  start ==> reading ==> ended ==> endEmitted
             || /\
           2 \/ || 3
             read
```

### 1. start ==> reading

start 状态变为 reading 状态，发生在第一次调用 ``read()`` 时

```js
// lib/_stream_readable.js

Readable.prototype.read = function(n) {
  debug('read', n);
  n = parseInt(n, 10);
  var state = this._readableState;
  var nOrig = n;

  if (n !== 0)
    state.emittedReadable = false;

  // 调用 read(0)时，如果缓存大于 highWaterMark 则直接触发 readable 事件
  if (n === 0 &&
      state.needReadable &&
      (state.length >= state.highWaterMark || state.ended)) {
    debug('read: emitReadable', state.length, state.ended);
    if (state.length === 0 && state.ended)
      endReadable(this);
    else
      emitReadable(this);
    return null;
  }

  // 计算可读数据量
  // n = NaN ==> 读取全部
  // n <= state.length ==> 读取 n
  // n > state.length ==> 读取 0，并使 Readable 从数据源读取数据
  // 
  // n > state.highWaterMark ==> 重新计算 highWaterMark，大小是大于 n 的最小 2^x
  n = howMuchToRead(n, state);

  // 当 Readable 已经读完时，调用 endReadable() ，结束 Readable
  if (n === 0 && state.ended) {
    if (state.length === 0)
      endReadable(this);
    return null;
  }

  // 判断是否应该从数据源读取数据
  // BEGIN
  var doRead = state.needReadable;
  debug('need readable', doRead);

  if (state.length === 0 || state.length - n < state.highWaterMark) {
    doRead = true;
    debug('length less than watermark', doRead);
  }
  // END

  if (state.ended || state.reading) {
    // 对于 ended 或 reading 状态的 Readable 是不需要读取数据的
    doRead = false;
    debug('reading or ended', doRead);
  } else if (doRead) {
    // 读取数据
    debug('do read');
    state.reading = true;
    state.sync = true;

    if (state.length === 0)
      state.needReadable = true;
    // 从数据源读取数据，可能是异步，也可能是同步
    this._read(state.highWaterMark);
    state.sync = false;
    // 因为 _read() 函数可能是异步的，也可能是同步的
    // 在同步情况下，需要重新确认可读长度
    if (!state.reading)
      n = howMuchToRead(nOrig, state);
  }

  // 获取数据
  var ret;
  if (n > 0) ret = fromList(n, state); // 从缓冲池中读取数据
  else ret = null;

  if (ret === null) {
    state.needReadable = true;
    n = 0;
  } else {
    state.length -= n;
  }

  // ...

  if (ret !== null)
    this.emit('data', ret);

  return ret;
};

// 必须实现的方法
Readable.prototype._read = function(n) {
  this.emit('error', new Error('_read() is not implemented'));
};

// 计算可读长度
function howMuchToRead(n, state) {
  if (n <= 0 || (state.length === 0 && state.ended))
    return 0;
  if (state.objectMode)
    return 1;
  if (n !== n) { // NaN
    if (state.flowing && state.length)
      return state.buffer.head.data.length;
    else
      return state.length;
  }

  if (n > state.highWaterMark)
    // 当需要数据大于 highWaterMark 时，调整 highWaterMark 大小到大于 n 的最小 2^x
    state.highWaterMark = computeNewHighWaterMark(n);
  if (n <= state.length)
    return n;
  // 缓冲池中数据不够
  if (!state.ended) {
    state.needReadable = true;
    return 0;
  }
  return state.length;
}
```

调用 ``read()`` 后，如果缓冲池中数据不够或读取后低于 higiWaterMark，则调用 ``_read()`` 来读取更多的数据，否则直接返回读取的数据

当期望数据量大于 highWaterMark 时，重新计算 highWaterMark，大小是大于期望数据量的最小 2^x

### 2. reading ==> read

调用 ``_read()`` 后，会异步或同步地将调用 ``push(chunk)``，将数据放入缓冲池，并使 Readable 从 reading 状态变为 read 状态

```js
// lib/_stream_readable.js

Readable.prototype.push = function(chunk, encoding) {
  var state = this._readableState;
  var skipChunkCheck;

  if (!state.objectMode) {
    if (typeof chunk === 'string') {
      encoding = encoding || state.defaultEncoding;
      // 如果指定编码与 Readable 编码不同，则将 chunk 使用指定编码解码为 Buffer
      if (encoding !== state.encoding) {
        chunk = Buffer.from(chunk, encoding);
        encoding = '';
      }
      // string 不需要检查
      skipChunkCheck = true;
    }
  } else {
    // object mode 的 Readable 也不需要检查
    skipChunkCheck = true;
  }

  return readableAddChunk(this, chunk, encoding, false, skipChunkCheck);
};

function readableAddChunk(stream, chunk, encoding, addToFront, skipChunkCheck) {
  var state = stream._readableState;
  if (chunk === null) { // 结束信号
    state.reading = false;
    onEofChunk(stream, state);
  } else {
    var er;
    if (!skipChunkCheck) // 检查 chunk 格式
      er = chunkInvalid(state, chunk);
    if (er) {
      stream.emit('error', er);
    } else if (state.objectMode || chunk && chunk.length > 0) {
      if (typeof chunk !== 'string' &&
          Object.getPrototypeOf(chunk) !== Buffer.prototype &&
          !state.objectMode) {
        chunk = Stream._uint8ArrayToBuffer(chunk);
      }

      if (addToFront) { // unshift() 的 hook
        if (state.endEmitted)
          stream.emit('error', new Error('stream.unshift() after end event'));
        else
          addChunk(stream, state, chunk, true); // 将数据添加到缓冲池中
      } else if (state.ended) {
        stream.emit('error', new Error('stream.push() after EOF'));
      } else {
        state.reading = false;
        if (state.decoder && !encoding) {
          chunk = state.decoder.write(chunk);
          if (state.objectMode || chunk.length !== 0)
            addChunk(stream, state, chunk, false); // 将数据添加到缓冲池中
          else // 会在 addChunk() 函数内部调用
            maybeReadMore(stream, state); // 将数据添加到缓冲池中
        } else {
          addChunk(stream, state, chunk, false); // 将数据添加到缓冲池中
        }
      }
    } else if (!addToFront) {
      state.reading = false;
    }
  }

  // return !state.ended &&  数据源还有数据
  //          (state.needReadable ||  需要更多数据
  //           state.length < state.highWaterMark ||  缓存小于 highWaterMark
  //           state.length === 0)
  return needMoreData(state);
}

function addChunk(stream, state, chunk, addToFront) {
  if (state.flowing && state.length === 0 && !state.sync) {
    // 对于 flow 模式的 Readable，直接触发 data 事件，并继续读取数据就行
    stream.emit('data', chunk);
    stream.read(0);
  } else {
    state.length += state.objectMode ? 1 : chunk.length;
    if (addToFront)
      state.buffer.unshift(chunk);
    else
      state.buffer.push(chunk);

    if (state.needReadable)
      emitReadable(stream);
  }

  // 在允许的情况下，读取数据直到 highWaterMark
  maybeReadMore(stream, state);
}
```

调用 ``push(chunk)`` 时，会将 chunk 放入缓冲池内，并改变 Readable 的状态。如果 Readable 处于 ended 状态，会报 ``stream.push() after EOF`` 错误

如果缓存小于 highWaterMark，返回 true，意味着需要写入更多的数据

### 3. read ==> reading

从 read 到 reading 状态，意味着需要读取更多的数据，即缓存小于 highWaterMark

缓存与 highWaterMark 的关系可以根据 ``push(chunk)`` 的返回值来判断，但是需要使用者手动处理。因此，为了方便使用，``addChunk()`` 函数会自动调用 ``maybeReadMore()`` 来异步读取数据。这样，即使单次 ``_read()`` 无法达到 highWaterMark，也可以通过多次异步读取，使数据流动起来

```js
// lib/_stream_readable.js

function maybeReadMore(stream, state) {
  if (!state.readingMore) {
    state.readingMore = true;
    process.nextTick(maybeReadMore_, stream, state);
  }
}

function maybeReadMore_(stream, state) {
  var len = state.length;
  while (!state.reading && !state.flowing && !state.ended &&
         state.length < state.highWaterMark) {
    debug('maybeReadMore read 0');
    stream.read(0);
    if (len === state.length) // 取不到数据就放弃
      break;
    else
      len = state.length;
  }
  state.readingMore = false;
}
```

在 ``maybeReadMore()`` 函数内，通过异步读取数据，直到 highWaterMark

那么为什么是异步读取数据呢？

因为，在 ``_read()`` 函数内，可能不止一次调用 ``push(chunk)``

如果是同步，``push(chunk)`` 后，因为没有达到 highWaterMark，会继续调用 ``read(0)``，发生第二次 ``_read()``。第二次 ``_read()`` 也可能导致第三次 ``_read()`` ，直到 highWaterMark

待整个调用完毕后，缓冲池内会有 highWaterMark * n（``_read()`` 内调用 ``push(chunk)`` 次数）的数据，而这与 highWaterMark 的设计是不符的

如果是异步，则可以等 ``_read()`` 执行完毕后，在 ``process.nextTick()`` 内再次调用 ``_read()`` 读取数据，不会发生上面的问题

### 4. reading ==> ended

当数据源读取完毕时，需要调用 ``push(null)`` 来通知 Rreadable 数据源已经读取完毕。``push(null)`` 函数内部会调用 ``onEofChunk()``

```js
// lib/_stream_readable.js

function onEofChunk(stream, state) {
  if (state.ended) return;
  if (state.decoder) {
    var chunk = state.decoder.end();
    if (chunk && chunk.length) {
      state.buffer.push(chunk);
      state.length += state.objectMode ? 1 : chunk.length;
    }
  }
  state.ended = true;

  // 触发 readable 事件，通知监听者来处理剩余数据
  emitReadable(stream);
}
```

``onEofChunk()`` 函数将 readable 标记为 ended 状态后，禁止再向缓冲池内 push 数据。此时，缓冲池内可能还有数据

### 5. ended ==> endEmitted

ended 状态的 Readable 内可能还有数据。因此，当数据全部被读取后，需要调用 ``endReadable()`` 来结束 Readable 

```js
// lib/_stream_readable.js

function endReadable(stream) {
  var state = stream._readableState;

  // state.length 一定是 0
  if (state.length > 0)
    throw new Error('"endReadable()" called on non-empty stream');

  if (!state.endEmitted) {
    state.ended = true;
    process.nextTick(endReadableNT, state, stream);
  }
}

function endReadableNT(state, stream) {
  // 防止中间调用 unshift(chunk)，向缓冲池中放入数据
  if (!state.endEmitted && state.length === 0) {
    state.endEmitted = true;
    stream.readable = false;
    stream.emit('end');
  }
}
```

调用 ``endReadable()`` 时，缓冲池一定为空。整个调用完成后，触发 end 事件，Readable 将不能再读取或写入（``push()`` / ``unshift()``）数据

## 总结

到这里，已经走完了 Readable 的整个 Birth-Death 过程

整个过程就如下面这个图:

```
         1           4         5
  start ==> reading ==> ended ==> endEmitted
             || /\
           2 \/ || 3
             read

1. read()
2. push(chunk)
3. maybeReadMore() ==> read(0)
4. push(null)
5. endReadable()
```

根据这个图还有代码，在脑袋里面，把 Readable 的模型运行一遍，就能了解它的实现了