# node 启动 - js 部分

> node 版本 8.x

node 进程启动时，首先执行 c / c++ 代码，然后 c / c++ 加载并执行 ``lib/internal/bootstrap_node.js`` 并给予一个 ``process`` 参数(运行上下文)

```js
// lib/internal/bootstrap_node.js 概览

// Hello, and welcome to hacking node.js!
//
// This file is invoked by node::LoadEnvironment in src/node.cc, and is
// responsible for bootstrapping the node.js core. As special caution is given
// to the performance of the startup process, many dependencies are invoked
// lazily.

'use strict';

// 这里的 process 对象来自 c / c++，属于原始数据
(function(process) {
  // ...

  startup();
})
```

加载 ``lib/internal/bootstrap_node.js`` 后，直接执行 ``startup()`` 函数

## startup()

```js
// lib/internal/bootstrap_node.js

  function startup() {
    // 下面几行代码使 process 具有 EventEmitter 的特性，比如说 on，emit
    // BEGIN 
    const EventEmitter = NativeModule.require('events');
    process._eventsCount = 0;

    const origProcProto = Object.getPrototypeOf(process);
    Object.setPrototypeOf(process, Object.create(EventEmitter.prototype, {
      constructor: Object.getOwnPropertyDescriptor(origProcProto, 'constructor')
    }));
    // 相当于 new EventEmitter() ，不过上下文是 process
    EventEmitter.call(process);
    // END

    // 一些初始化操作
    // BEGIN
    setupProcessObject();
    // do this good and early, since it handles errors.
    setupProcessFatal();
    setupProcessICUVersions();
    setupGlobalVariables();
    if (!process._noBrowserGlobals) {
      setupGlobalTimeouts();
      setupGlobalConsole();
    }
    // END

    // process 对象的初始化操作
    // BEGIN
    // 这里的 internal/process 模块用于初始化 process 
    const _process = NativeModule.require('internal/process');
    _process.setup_hrtime();
    _process.setup_cpuUsage();
    _process.setupMemoryUsage();
    _process.setupConfig(NativeModule._source);
    NativeModule.require('internal/process/warning').setup();
    NativeModule.require('internal/process/next_tick').setup();
    NativeModule.require('internal/process/stdio').setup();
    _process.setupKillAndExit();
    _process.setupSignalHandlers();
    if (global.__coverage__)
      NativeModule.require('internal/process/write-coverage').setup();

    if (process.argv[1] !== '--debug-agent')
      _process.setupChannel();

    _process.setupRawDebug();
    
    NativeModule.require('internal/url');

    Object.defineProperty(process, 'argv0', {
      enumerable: true,
      configurable: false,
      value: process.argv[0]
    });
    process.argv[0] = process.execPath;
    // ...
    // END

    // 下面的 if-else 用来判断 node 的运行模式，我们只关注 node xx.js 的运行模式

    // if () {
    // ...
    } else {
      // 执行用户代码

      // cluster 模块的 hook
      if (process.argv[1] && process.env.NODE_UNIQUE_ID) {
        const cluster = NativeModule.require('cluster');
        cluster._setupWorker();

        // Make sure it's not accidentally inherited by child processes.
        delete process.env.NODE_UNIQUE_ID;
      }

      if (process._eval != null && !process._forceRepl) {
        // ...
      } else if (process.argv[1] && process.argv[1] !== '-') {
        // node app.js

        // make process.argv[1] into a full path
        const path = NativeModule.require('path');
        // 变为绝对路径
        process.argv[1] = path.resolve(process.argv[1]);

        const Module = NativeModule.require('module');

        // check if user passed `-c` or `--check` arguments to Node.
        if (process._syntax_check_only != null) {
          const fs = NativeModule.require('fs');
          // read the source
          const filename = Module._resolveFilename(process.argv[1]);
          var source = fs.readFileSync(filename, 'utf-8');
          checkScriptSyntax(source, filename);
          process.exit(0);
        }

        preloadModules();
        Module.runMain();
      } else {
        // REPL 或其他
      }
    }
  }
```

``startup()`` 最后调用 ``Module.runMain()`` 函数来加载并执行用户代码。在执行 ``startup()`` 函数的过程中，多次用到了 ``NativeModule.require`` 来加载模块

## NativeModule

``NativeModule.require`` 是专门用来加载 node 原生模块的

```js
// lib/internal/bootstrap_node.js

  function NativeModule(id) {
    this.filename = `${id}.js`;
    this.id = id;
    this.exports = {};
    this.loaded = false;
    this.loading = false;
  }

  NativeModule._source = process.binding('natives');
  NativeModule._cache = {};

  NativeModule.require = function(id) {
    if (id === 'native_module') {
      return NativeModule;
    }

    const cached = NativeModule.getCached(id);
    if (cached && (cached.loaded || cached.loading)) {
      return cached.exports;
    }

    if (!NativeModule.exists(id)) {
      // Model the error off the internal/errors.js model, but
      // do not use that module given that it could actually be
      // the one causing the error if there's a bug in Node.js
      const err = new Error(`No such built-in module: ${id}`);
      err.code = 'ERR_UNKNOWN_BUILTIN_MODULE';
      err.name = 'Error [ERR_UNKNOWN_BUILTIN_MODULE]';
      throw err;
    }

    process.moduleLoadList.push(`NativeModule ${id}`);

    const nativeModule = new NativeModule(id);

    nativeModule.cache();
    nativeModule.compile();

    return nativeModule.exports;
  };

  NativeModule.getCached = function(id) {
    return NativeModule._cache[id];
  };

  NativeModule.exists = function(id) {
    return NativeModule._source.hasOwnProperty(id);
  };

  // ...

  NativeModule.wrap = function(script) {
    return NativeModule.wrapper[0] + script + NativeModule.wrapper[1];
  };

  NativeModule.wrapper = [
    '(function (exports, require, module, __filename, __dirname) { ',
    '\n});'
  ];

  NativeModule.prototype.compile = function() {
    var source = NativeModule.getSource(this.id);
    source = NativeModule.wrap(source);

    this.loading = true;

    try {
      const fn = runInThisContext(source, {
        filename: this.filename,
        lineOffset: 0,
        displayErrors: true
      });
      fn(this.exports, NativeModule.require, this, this.filename);

      this.loaded = true;
    } finally {
      this.loading = false;
    }
  };

  NativeModule.prototype.cache = function() {
    NativeModule._cache[this.id] = this;
  };
```

``NativeModule`` 有几个重要的属性和方法:
  * ``id``: ``NativeModule`` 的标识符，例如 ``events``，``internal/process``
  * ``filename``:  ``NativeModule`` 对应源码文件
  * ``exports``: 默认值是 ``{}``
  * ``loaded`` / ``loading``: ``NativeModule`` 状态
  * ``_cache``: 简单的模块缓存
  * ``_source``: 模块源码资源
  * ``require()``: 先查询缓存，缓存没有则新建 ``NativeModule`` 并编译，返回 ``exports``
  * ``wrap()/wrapper``: ``wrapper`` 是对模块上下文的包裹，如下:
    ```js
    (function (exports, require, module, __filename, __dirname) { 
      xxx
    });
    ```
  * ``compile()``: 将模块源码用 ``wrapper`` 包裹后，使用 ``runInThisContext()``（类似 ``eval()``）生成 ``js`` 函数，再执行之

## Module.runMain()

node 启动完成后，调用 ``Module.runMain()``，源码如下:

```js
// bootstrap main module.
Module.runMain = function() {
  // Load the main module--the command line argument.
  Module._load(process.argv[1], null, true);
  // Handle any nextTicks added in the first tick of the program
  process._tickCallback();
};
```

``Module._load()`` 加载并执行用户代码。至此 ``node启动-js部分`` 已经全部完成，后续模块加载部分，见 [require背后](./require背后.md)