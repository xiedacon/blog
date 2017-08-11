---
title: Node.js源码解析-pipe实现
date: 2017-08-11
tags: Node.js源码解析
---

# Node.js源码解析-pipe实现

从前面两篇文章，我们了解到。想要把 Readable 的数据写到 Writable，就必须先手动的将数据读入内存，然后写入 Writable。换句话说，每次传递数据时，都需要写如下的模板代码

```js
readable.on('readable', (err) => {
  if(err) throw err

  writable.write(readable.read())
})
```

为了方便使用，Node.js 提供了 ``pipe()`` 方法，让我们可以优雅的传递数据

```js
readable.pipe(writable)
```

现在，就让我们来看看它是如何实现的吧

## pipe

首先需要先调用 Readable 的 ``pipe()`` 方法

```js
// lib/_stream_readable.js

Readable.prototype.pipe = function(dest, pipeOpts) {
  var src = this;
  var state = this._readableState;

  // 记录 Writable
  switch (state.pipesCount) {
    case 0:
      state.pipes = dest;
      break;
    case 1:
      state.pipes = [state.pipes, dest];
      break;
    default:
      state.pipes.push(dest);
      break;
  }
  state.pipesCount += 1;

  // ...

    src.once('end', endFn);

  dest.on('unpipe', onunpipe);
  
  // ...

  dest.on('drain', ondrain);

  // ...

  src.on('data', ondata);

  // ...

  // 保证 error 事件触发时，onerror 首先被执行
  prependListener(dest, 'error', onerror);

  // ...

  dest.once('close', onclose);
  
  // ...

  dest.once('finish', onfinish);

  // ...

  // 触发 Writable 的 pipe 事件
  dest.emit('pipe', src);

  // 将 Readable 改为 flow 模式
  if (!state.flowing) {
    debug('pipe resume');
    src.resume();
  }

  return dest;
};
```

执行 ``pipe()`` 函数时，首先将 Writable 记录到 ``state.pipes`` 中，然后绑定相关事件，最后如果 Readable 不是 flow 模式，就调用 ``resume()`` 将 Readable 改为 flow 模式

## 传递数据

Readable 从数据源获取到数据后，触发 data 事件，执行 ``ondata()``

``ondata()`` 相关代码：

```js
// lib/_stream_readable.js

  // 防止在 dest.write(chunk) 内调用 src.push(chunk) 造成 awaitDrain 重复增加，awaitDrain 不能清零，Readable 卡住的情况
  // 详情见 https://github.com/nodejs/node/issues/7278
  var increasedAwaitDrain = false;
  function ondata(chunk) {
    debug('ondata');
    increasedAwaitDrain = false;
    var ret = dest.write(chunk);
    if (false === ret && !increasedAwaitDrain) {
      // 防止在 dest.write() 内调用 src.unpipe(dest)，导致 awaitDrain 不能清零，Readable 卡住的情况
      if (((state.pipesCount === 1 && state.pipes === dest) ||
           (state.pipesCount > 1 && state.pipes.indexOf(dest) !== -1)
          ) && 
          !cleanedUp) {
        debug('false write response, pause', src._readableState.awaitDrain);
        src._readableState.awaitDrain++;
        increasedAwaitDrain = true;
      }
      // 进入 pause 模式
      src.pause();
    }
  }
```

在 ``ondata(chunk)`` 函数内，通过 ``dest.write(chunk)`` 将数据写入 Writable

此时，在 ``_write()`` 内部可能会调用 ``src.push(chunk)`` 或使其 unpipe，这会导致 awaitDrain 多次增加，不能清零，Readable 卡住

当不能再向 Writable 写入数据时，Readable 会进入 pause 模式，直到所有的 drain 事件触发

触发 drain 事件，执行 ``ondrain()``

```js
// lib/_stream_readable.js

  var ondrain = pipeOnDrain(src);

  function pipeOnDrain(src) {
    return function() {
      var state = src._readableState;
      debug('pipeOnDrain', state.awaitDrain);
      if (state.awaitDrain)
        state.awaitDrain--;
      // awaitDrain === 0，且有 data 监听器
      if (state.awaitDrain === 0 && EE.listenerCount(src, 'data')) {
        state.flowing = true;
        flow(src);
      }
    };
  }
```

每个 drain 事件触发时，都会减少 awaitDrain，直到 awaitDrain 为 0。此时，调用 ``flow(src)``，使 Readable 进入 flow 模式

到这里，整个数据传递循环已经建立，数据会顺着循环源源不断的流入 Writable，直到所有数据写入完成

## unpipe

不管写入过程中是否出现错误，最后都会执行 ``unpipe()``

```js
// lib/_stream_readable.js

// ...

  function unpipe() {
    debug('unpipe');
    src.unpipe(dest);
  }

// ...

Readable.prototype.unpipe = function(dest) {
  var state = this._readableState;
  var unpipeInfo = { hasUnpiped: false };

  // 啥也没有
  if (state.pipesCount === 0)
    return this;

  // 只有一个
  if (state.pipesCount === 1) {
    if (dest && dest !== state.pipes)
      return this;
    // 没有指定就 unpipe 所有
    if (!dest)
      dest = state.pipes;

    state.pipes = null;
    state.pipesCount = 0;
    state.flowing = false;
    if (dest)
      dest.emit('unpipe', this, unpipeInfo);
    return this;
  }

  // 没有指定就 unpipe 所有
  if (!dest) {
    var dests = state.pipes;
    var len = state.pipesCount;
    state.pipes = null;
    state.pipesCount = 0;
    state.flowing = false;

    for (var i = 0; i < len; i++)
      dests[i].emit('unpipe', this, unpipeInfo);
    return this;
  }

  // 找到指定 Writable，并 unpipe
  var index = state.pipes.indexOf(dest);
  if (index === -1)
    return this;

  state.pipes.splice(index, 1);
  state.pipesCount -= 1;
  if (state.pipesCount === 1)
    state.pipes = state.pipes[0];

  dest.emit('unpipe', this, unpipeInfo);

  return this;
};
```

``Readable.prototype.unpipe()`` 函数会根据 ``state.pipes`` 属性和 dest 参数，选择执行策略。最后会触发 dest 的 unpipe 事件

unpipe 事件触发后，调用 ``onunpipe()``，清理相关数据

```js
// lib/_stream_readable.js

  function onunpipe(readable, unpipeInfo) {
    debug('onunpipe');
    if (readable === src) {
      if (unpipeInfo && unpipeInfo.hasUnpiped === false) {
        unpipeInfo.hasUnpiped = true;
        // 清理相关数据
        cleanup();
      }
    }
  }
```

## End

在整个 pipe 的过程中，Readable 是主动方 ( 负责整个 pipe 过程：包括数据传递、unpipe 与异常处理 )，Writable 是被动方 ( 只需要触发 drain 事件 )

总结一下 pipe 的过程：
* 首先执行 ``readbable.pipe(writable)``，将 readable 与 writable 对接上
* 当 readable 中有数据时，``readable.emit('data')``，将数据写入 writable
* 如果 ``writable.write(chunk)`` 返回 false，则进入 pause 模式，等待 drain 事件触发
* drain 事件全部触发后，再次进入 flow 模式，写入数据
* 不管数据写入完成或发生中断，最后都会调用 ``unpipe()``
* ``unpipe()`` 调用 ``Readable.prototype.unpipe()``，触发 dest 的 unpipe 事件，清理相关数据

参考：
  * [https://github.com/nodejs/node/blob/master/lib/_stream_readable.js](https://github.com/nodejs/node/blob/master/lib/_stream_readable.js)
  * [https://github.com/nodejs/node/blob/master/lib/_stream_writable.js](https://github.com/nodejs/node/blob/master/lib/_stream_writable.js)
