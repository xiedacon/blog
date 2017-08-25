---
title: Node.js源码解析-Buffer的8k池实现
date: 2017-08-25
tags: Node.js源码解析
---

# Node.js源码解析-Buffer的8k池实现

在 Node.js 中，对于大文件一般是以 Buffer 形式存储，相比于字符串，Buffer 可以免去 ``decode`` / ``encode`` 过程，节省 CPU 成本

说到 Buffer 就不得不提到 Buffer 的 8k 池，那么，下面就让我们来看看 8k 池是如何实现的吧

## 8k池实现

在 ``Node.js v8.4.0`` 中，可以通过以下方法来获取一个 Buffer 实例：
  * ``new Buffer()`` ( 不推荐 )
  * ``Buffer.from()``
  * ``Buffer.alloc()``
  * ``Buffer.allocUnsafe()``
  * ``Buffer.allocUnsafeSlow()``

从命名上来看，``Buffer.allocUnsafe()`` 和 ``Buffer.allocUnsafeSlow()`` 都是不安全的，有泄漏内存中敏感信息的危险

unsafe 的问题放到后面说，先看看如何获取一个 Buffer 实例

![创建 Buffer](http://upload-images.jianshu.io/upload_images/2395997-4d09045ce5d5807f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以看出，只有 ``allocate()`` 和 ``fromString()`` 两个函数直接与8k池相关

```js
// lib/buffer.js

function allocate(size) {
  if (size <= 0) {
    return new FastBuffer();
  }
  if (size < (Buffer.poolSize >>> 1)) { // < 4k
    if (size > (poolSize - poolOffset)) // 大于剩余容量
      createPool();
    var b = new FastBuffer(allocPool, poolOffset, size);
    poolOffset += size;
    alignPool();
    return b;
  } else { // > 4k
    return createUnsafeBuffer(size);
  }
}

function fromString(string, encoding) {
  // ...
  if (length >= (Buffer.poolSize >>> 1)) // > 4k
    return binding.createFromString(string, encoding);

  if (length > (poolSize - poolOffset)) // 大于剩余容量
    createPool();
  var b = new FastBuffer(allocPool, poolOffset, length);
  const actual = b.write(string, encoding);
  if (actual !== length) { // byteLength() 可能会估计错误，尽管可能性很小
    b = new FastBuffer(allocPool, poolOffset, actual);
  }
  poolOffset += actual;
  alignPool();
  return b;
}
```

``allocate()`` 和 ``fromString()`` 都是分为大于 4k 和小于 4k 两种情况来处理

小于 4k 时，先检查8k池的剩余容量，如果大于剩余容量则直接创建一个新的8k池，然后修正 poolOffset，最后调用 ``alignPool()``

```js
// lib/buffer.js

Buffer.poolSize = 8 * 1024;
var poolSize, poolOffset, allocPool;

function createUnsafeArrayBuffer(size) {
  // ...
  return new ArrayBuffer(size);
  // ...
}

function createPool() {
  poolSize = Buffer.poolSize; // 8k
  allocPool = createUnsafeArrayBuffer(poolSize);
  poolOffset = 0;
}
createPool();

function alignPool() {
  if (poolOffset & 0x7) { // 进行校准，只能为 8 的倍数
    poolOffset |= 0x7; // xxx111
    poolOffset++; // xx(x+1)000
  }
}
```

通过调用 ``alignPool()`` 来校准 poolOffset，poolOffset 只能为 8 的倍数，换句话说，每次至少使用 8 个字节内存

8k池容量不够时，调用 ``createPool()``，创建一个新的8k池

``createPool()`` 内部调用 ``createUnsafeArrayBuffer()`` 来获取一个对应大小的 ArrayBuffer 实例

关于 ArrayBuffer，这里引用 MDN 的介绍：

  > The ArrayBuffer object is used to represent a generic, fixed-length raw binary data buffer

因为 ArrayBuffer 是 ``raw binary data``，所以它是不安全的，存在泄漏内存中敏感信息的危险

### Why Unsafe ?

从图中我们知道，一共有 4 种方法来获得一个 Buffer 实例，它们之中，有的是 unsafe 的，有的不是

> ``new Buffer()`` 依赖 ``Buffer.from()`` 和 ``Buffer.alloc()`` 不算一种

下面让我们来看看为什么有的是 unsafe 的

```js
// lib/buffer.js

// safe
Buffer.from = function(value, encodingOrOffset, length) {
  if (typeof value === 'string')
    return fromString(value, encodingOrOffset); // safe

  if (isAnyArrayBuffer(value))
    return fromArrayBuffer(value, encodingOrOffset, length); // safe

  var b = fromObject(value); // safe
  // ...
};

function fromString(string, encoding) {
  // ...
  var b = new FastBuffer(allocPool, poolOffset, length);
  const actual = b.write(string, encoding);
  // ...
}

function fromArrayBuffer(obj, byteOffset, length) {
  // ...
  return new FastBuffer(obj, byteOffset, length);
}

function fromObject(obj) {
  if (isUint8Array(obj)) {
    const b = allocate(obj.length);
    // ...
    binding.copy(obj, b, 0, 0, obj.length);
  }

  if (obj != null) {
    // ...
      return fromArrayLike(obj);
    // ...
  }
}

function fromArrayLike(obj) {
  // ...
  const b = allocate(length);
  for (var i = 0; i < length; i++)
    b[i] = obj[i];
  return b;
}

// unsafe
Buffer.allocUnsafe = function(size) {
  return allocate(size);
};

// unsafe
Buffer.allocUnsafeSlow = function(size) {
  return createUnsafeBuffer(size);
};

function createUnsafeBuffer(size) {
  return new FastBuffer(createUnsafeArrayBuffer(size));
}

// safe
Buffer.alloc = function(size, fill, encoding) {
  assertSize(size);
  if (size > 0 && fill !== undefined) {
    // ...
    return createUnsafeBuffer(size).fill(fill, encoding); 
  }
  return new FastBuffer(size);
};
```

可以看出:
  * ``Buffer.from()`` 和 ``Buffer.alloc()``: 取到原始 buffer 后，对原始数据进行了替换，所以它们是 safe 的
  * ``Buffer.allocUnsafe()`` 和 ``Buffer.allocUnsafeSlow()``: 直接使用原始 buffer，所以它们是 unsafe 的

## End

* ``new Buffer()``: 依赖 ``Buffer.from()`` 和 ``Buffer.alloc()``
* ``Buffer.from()``
  * ArrayBuffer: 直接使用 ArrayBuffer 创建 FastBuffer
  * String: 小于 4k 使用8k池，大于 4k 调用 ``binding.createFromString()``
  * Object: 小于 4k 使用8k池，大于 4k 调用 ``createUnsafeBuffer()``
* ``Buffer.alloc()``: 需要 fill buffer，用给定字符填充，否则用 0 填充
* ``Buffer.allocUnsafe()``: 小于 4k 使用8k池，大于 4k 调用 ``createUnsafeBuffer()``
* ``Buffer.allocUnsafeSlow()``: 调用 ``createUnsafeBuffer()``

参考：
  * [https://github.com/nodejs/node/blob/master/lib/buffer.js](https://github.com/nodejs/node/blob/master/lib/buffer.js)