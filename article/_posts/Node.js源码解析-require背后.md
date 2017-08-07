---
title: Node.js源码解析-require背后
date: 2017-07-03
tags: Node.js源码解析
---
# Node.js源码解析-require背后

在编写 Node.js 应用的过程中，我们或多或少的都写过类似 ``const xxx = require('xxx')`` 的代码，其作用是引入模块。不知大家有没有想过，这段代码是如何确定我们要引入的模块？又是以怎样的上下文来执行模块代码的呢？

让我们来翻开 Node.js 源码，先找到 ``lib/module.js`` 中的 ``Module.prototype.require()`` 函数

```js
// lib/module.js

Module.prototype.require = function(path) {
  assert(path, 'missing path');
  assert(typeof path === 'string', 'path must be a string');
  return Module._load(path, this, /* isMain */ false);
};

// Check the cache for the requested file.
// 1. If a module already exists in the cache: return its exports object.
// 2. If the module is native: call `NativeModule.require()` with the
//    filename and return the result.
// 3. Otherwise, create a new module for the file and save it to the cache.
//    Then have it load  the file contents before returning its exports
//    object.
Module._load = function(request, parent, isMain) {
  if (parent) {
    debug('Module._load REQUEST %s parent: %s', request, parent.id);
  }

  var filename = Module._resolveFilename(request, parent, isMain);

  var cachedModule = Module._cache[filename];
  if (cachedModule) {
    return cachedModule.exports;
  }

  if (NativeModule.nonInternalExists(filename)) {
    debug('load native module %s', request);
    return NativeModule.require(filename);
  }

  var module = new Module(filename, parent);

  if (isMain) {
    process.mainModule = module;
    module.id = '.';
  }

  Module._cache[filename] = module;

  tryModuleLoad(module, filename);

  return module.exports;
};
```

``Module.prototype.require()`` 对传入的 path 简单断言后调用 ``Module._load()`` 来导入模块

``Module._load()`` 的执行思路如下:
  * 查找模块: ``Module._resolveFilename()``
  * 存在缓存: 返回 ``cachedModule.exports``
  * 是内置模块: 见 [NativeModule.require()](./Node.js源码解析-启动-js部分.md#nativemodule)
  * 加载模块: ``tryModuleLoad()``

因此，``Module.prototype.require()`` 的源码可以分为两大块: 查找模块和加载模块

## 查找模块

查找模块的关键在于定位模块的具体路径，这个功能由 ``Module._resolveFilename()`` 函数实现

```js
// lib/module.js

Module._resolveFilename = function(request, parent, isMain) {
  if (NativeModule.nonInternalExists(request)) {
    return request;
  }

  // 可能存在要加载模块的目录
  var paths = Module._resolveLookupPaths(request, parent, true);

  // 具体的模块路径
  var filename = Module._findPath(request, paths, isMain);
  if (!filename) {
    var err = new Error(`Cannot find module '${request}'`);
    err.code = 'MODULE_NOT_FOUND';
    throw err;
  }
  return filename;
};
```

``Module._resolveFilename()`` 函数先调用 ``Module._resolveLookupPaths()`` 计算出模块可能存在的目录，然后调用 ``Module._findPath()`` 得到模块路径

### Module._resolveLookupPaths

通过调用 ``Module._resolveLookupPaths()`` 函数可以计算出模块可能存在的目录，在调用时存在三种情况:
  * 直接 ``require('xxx')``: 需要递归查询路径上的 ``node_modules`` 目录和全局 ``node_modules`` 目录
  * 通过 ``Module.runMain()`` 或 ``--eval`` 参数: 返回执行命令行的目录
  * 使用相对 / 绝对路径导入: 这时，直接返回父模块目录即可

```js
// lib/module.js

Module._resolveLookupPaths = function(request, parent, newReturn) {
  if (NativeModule.nonInternalExists(request)) {
    debug('looking for %j in []', request);
    return (newReturn ? null : [request, []]);
  }

  // 类似 require('xxx')
  if (request.length < 2 ||
      request.charCodeAt(0) !== 46 ||       // 非 . 开头
      (request.charCodeAt(1) !== 46         // 非 .. 开头
        && request.charCodeAt(1) !== 47)) { // 非 / 开头
    var paths = modulePaths;
    if (parent) {
      if (!parent.paths)
        paths = parent.paths = [];
      else
        paths = parent.paths.concat(paths);
    }

    // Maintain backwards compat with certain broken uses of require('.')
    // by putting the module's directory in front of the lookup paths.
    if (request === '.') {
      if (parent && parent.filename) {
        paths.unshift(path.dirname(parent.filename));
      } else {
        paths.unshift(path.resolve(request));
      }
    }

    debug('looking for %j in %j', request, paths);
    return (newReturn ? (paths.length > 0 ? paths : null) : [request, paths]);
  }

  // 执行 Module.runMain() 进入，此时 request 是绝对路径

  // with --eval, parent.id is not set and parent.filename is null
  if (!parent || !parent.id || !parent.filename) {
    // make require('./path/to/foo') work - normally the path is taken
    // from realpath(__filename) but with eval there is no filename
    var mainPaths = ['.'].concat(Module._nodeModulePaths('.'), modulePaths);

    debug('looking for %j in %j', request, mainPaths);
    return (newReturn ? mainPaths : [request, mainPaths]);
  }

  // ...

  var parentDir = [path.dirname(parent.filename)];
  debug('looking for %j in %j', id, parentDir);
  return (newReturn ? parentDir : [id, parentDir]);
};
```

### Module._findPath

使用 ``Module._resolveLookupPaths()`` 函数找到模块可能存在的目录后，调用 ``Module._findPath()`` 函数，递归查找模块

``Module._findPath()`` 函数在查找模块时，存在以下几种情况:
  * ``require`` 的是文件 ==> ``.js`` / ``.json`` / ``.node``
  * ``require`` 的文件夹 ==> 找 ``index.js`` / ``index.json`` / ``index.node``
  * ``require`` 的包 ==> 找 ``package.json``

```js
// lib/module.js

Module._findPath = function(request, paths, isMain) {
  if (path.isAbsolute(request)) {
    paths = [''];
  } else if (!paths || paths.length === 0) {
    return false;
  }

  // 计算 cacheKey
  // 对于同一模块，每个目录的 cacheKey 不同
  var cacheKey = request + '\x00' +
                (paths.length === 1 ? paths[0] : paths.join('\x00'));
  var entry = Module._pathCache[cacheKey];
  // 有缓存就走缓存
  if (entry)
    return entry;

  var exts;
  var trailingSlash = request.length > 0 &&
                      request.charCodeAt(request.length - 1) === 47; // /

  // For each path
  for (var i = 0; i < paths.length; i++) {
    const curPath = paths[i];
    // 不存在就跳过
    if (curPath && stat(curPath) < 1) continue;
    var basePath = path.resolve(curPath, request);
    var filename;

    var rc = stat(basePath);
    if (!trailingSlash) {
      if (rc === 0) {  // 文件
        if (preserveSymlinks && !isMain) {
          filename = path.resolve(basePath);
        } else {
          filename = toRealPath(basePath);
        }
      } else if (rc === 1) {  // 目录或 package
        if (exts === undefined)
          exts = Object.keys(Module._extensions);
        // 如果是目录，则 filename 为 false
        filename = tryPackage(basePath, exts, isMain);
      }

      // 对应是文件但未给出文件后缀的情况
      if (!filename) {
        // try it with each of the extensions
        if (exts === undefined)
          exts = Object.keys(Module._extensions);
        filename = tryExtensions(basePath, exts, isMain);
      }
    }

    if (!filename && rc === 1) {  // 目录或 package
      if (exts === undefined)
        exts = Object.keys(Module._extensions);
      // 如果是目录，则 filename 为 false
      filename = tryPackage(basePath, exts, isMain);
    }

    if (!filename && rc === 1) {  // 目录
      // try it with each of the extensions at "index"
      if (exts === undefined)
        exts = Object.keys(Module._extensions);
      filename = tryExtensions(path.resolve(basePath, 'index'), exts, isMain);
    }

    if (filename) {
      // Warn once if '.' resolved outside the module dir
      if (request === '.' && i > 0) {
        if (!warned) {
          warned = true;
          process.emitWarning(
            'warning: require(\'.\') resolved outside the package ' +
            'directory. This functionality is deprecated and will be removed ' +
            'soon.',
            'DeprecationWarning', 'DEP0019');
        }
      }

      Module._pathCache[cacheKey] = filename;
      return filename;
    }
  }
  return false;
};
```

上面的 for 循环内有许多重复代码，可以优化为:

```js
  var exts = Object.keys(Module._extensions);
  var trailingSlash = request.length > 0 &&
                      request.charCodeAt(request.length - 1) === 47; // /

  for (var i = 0; i < paths.length; i++) {
    const curPath = paths[i];
    // 不存在就跳过
    if (curPath && stat(curPath) < 1) continue;
    var basePath = path.resolve(curPath, request);
    var filename;

    var rc = stat(basePath);
    if (!trailingSlash && rc !== 1) {  // 文件或啥也没有
      if (rc === 0) {  // 文件
        if (preserveSymlinks && !isMain) {
          filename = path.resolve(basePath);
        } else {
          filename = toRealPath(basePath);
        }
      } else {  // 对应是文件但未给出文件后缀的情况
        filename = tryExtensions(basePath, exts, isMain);
      }
    }

    if(!filename && rc === 1){  // 目录或 package
      filename = tryPackage(basePath, exts, isMain);

      // 如果是目录，则 filename 为 false
      if (!filename) {
        // try it with each of the extensions at "index"
        filename = tryExtensions(path.resolve(basePath, 'index'), exts, isMain);
      }
    }

    if (filename) {
      // Warn once if '.' resolved outside the module dir
      if (request === '.' && i > 0) {
        if (!warned) {
          warned = true;
          process.emitWarning(
            'warning: require(\'.\') resolved outside the package ' +
            'directory. This functionality is deprecated and will be removed ' +
            'soon.',
            'DeprecationWarning', 'DEP0019');
        }
      }

      Module._pathCache[cacheKey] = filename;
      return filename;
    }
  }
```

``Module._findPath()`` 函数内部依赖下面几个方法:

```js
// lib/module.js

function tryPackage(requestPath, exts, isMain) {
  // 读取目录下的 package.json 并返回 package.main，没有则返回 false
  var pkg = readPackage(requestPath);

  if (!pkg) return false;

  var filename = path.resolve(requestPath, pkg);
  return tryFile(filename, isMain) || // package.main 有后缀
         tryExtensions(filename, exts, isMain) || // package.main 没有后缀
         tryExtensions(path.resolve(filename, 'index'), exts, isMain); // package.main 不存在则默认加载 index.js / index.json / index.node
}

function readPackage(requestPath) {
  const entry = packageMainCache[requestPath];
  if (entry)
    return entry;

  const jsonPath = path.resolve(requestPath, 'package.json');
  const json = internalModuleReadFile(path._makeLong(jsonPath));

  // 没有 package.json，说明不是一个包
  if (json === undefined) {
    return false;
  }

  try {
    var pkg = packageMainCache[requestPath] = JSON.parse(json).main;
  } catch (e) {
    e.path = jsonPath;
    e.message = 'Error parsing ' + jsonPath + ': ' + e.message;
    throw e;
  }
  return pkg;
}

// given a path check a the file exists with any of the set extensions
function tryExtensions(p, exts, isMain) {
  for (var i = 0; i < exts.length; i++) {
    const filename = tryFile(p + exts[i], isMain);

    if (filename) {
      return filename;
    }
  }
  return false;
}

// check if the file exists and is not a directory
// if using --preserve-symlinks and isMain is false,
// keep symlinks intact, otherwise resolve to the
// absolute realpath.
function tryFile(requestPath, isMain) {
  const rc = stat(requestPath);
  if (preserveSymlinks && !isMain) {
    return rc === 0 && path.resolve(requestPath);
  }
  return rc === 0 && toRealPath(requestPath);
}

function toRealPath(requestPath) {
  return fs.realpathSync(requestPath, {
    [internalFS.realpathCacheKey]: realpathCache
  });
}
```

通过以上步骤，我们找到了对应的模块文件，下面开始加载模块

## 加载模块

通过 ``Module._resolveFilename()`` 函数找到具体的模块文件路径后，就可以开始加载模块了

``Module._load()`` 调用 ``tryModuleLoad()`` 函数来加载模块。``tryModuleLoad()`` 则将 ``module.load()`` 函数（ 真正加载模块的函数 ）包裹在一个 try-finally 块中


```js
// lib/module.js

function tryModuleLoad(module, filename) {
  var threw = true;
  try {
    module.load(filename);
    threw = false;
  } finally {
    if (threw) {
      delete Module._cache[filename];
    }
  }
}
```

### Module.prototype.load()

对于 ``Module.prototype.load()`` 函数来说，模块文件有三种类型:
  * ``.js``: 读取文件然后调用 ``module._compile()`` 编译执行，这是默认情况
  * ``.json``: 作为 ``json`` 文件读取
  * ``.node``: 直接执行编译后的二进制文件

```js
// lib/module.js

// Given a file name, pass it to the proper extension handler.
Module.prototype.load = function(filename) {
  debug('load %j for module %j', filename, this.id);

  assert(!this.loaded);
  this.filename = filename;
  // 全局 node_modules 和文件路径上所有的 node_modules
  this.paths = Module._nodeModulePaths(path.dirname(filename));

  // 通过文件扩展名确定加载方式，默认采用 .js 的方式加载
  var extension = path.extname(filename) || '.js';
  if (!Module._extensions[extension]) extension = '.js';
  Module._extensions[extension](this, filename);
  this.loaded = true;
};

// Native extension for .js
Module._extensions['.js'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  module._compile(internalModule.stripBOM(content), filename);
};

// Native extension for .json
Module._extensions['.json'] = function(module, filename) {
  var content = fs.readFileSync(filename, 'utf8');
  try {
    module.exports = JSON.parse(internalModule.stripBOM(content));
  } catch (err) {
    err.message = filename + ': ' + err.message;
    throw err;
  }
};

//Native extension for .node
Module._extensions['.node'] = function(module, filename) {
  return process.dlopen(module, path._makeLong(filename));
};

// Run the file contents in the correct scope or sandbox. Expose
// the correct helper variables (require, module, exports) to
// the file.
// Returns exception, if any.
Module.prototype._compile = function(content, filename) {

  content = internalModule.stripShebang(content);

  // 使用下面这个结构包裹
  // (function (exports, require, module, __filename, __dirname) {
  //   ...
  // });
  var wrapper = Module.wrap(content);

  // 在沙箱内生成函数对象
  var compiledWrapper = vm.runInThisContext(wrapper, {
    filename: filename,
    lineOffset: 0,
    displayErrors: true
  });

  // ...

  var dirname = path.dirname(filename);
  var require = internalModule.makeRequireFunction(this);
  var depth = internalModule.requireDepth;
  if (depth === 0) stat.cache = new Map();
  var result;
  if (inspectorWrapper) {
    result = inspectorWrapper(compiledWrapper, this.exports, this.exports,
                              require, this, filename, dirname);
  } else {
    result = compiledWrapper.call(this.exports, this.exports, require, this,
                                  filename, dirname);
  }
  if (depth === 0) stat.cache = null;
  return result;
};
```

从上面的代码可以看出，执行模块时，模块中的 require，不是 ``module.js`` 中的 require，而是由 ``internalModule.makeRequireFunction()`` 生成的一个新的函数对象，其隐藏了 module 的实现细节，方便使用

```js
// lib/internal/module.js

// Invoke with makeRequireFunction(module) where |module| is the Module object
// to use as the context for the require() function.
function makeRequireFunction(mod) {
  const Module = mod.constructor;

  function require(path) {
    try {
      exports.requireDepth += 1;
      return mod.require(path);
    } finally {
      exports.requireDepth -= 1;
    }
  }

  function resolve(request) {
    return Module._resolveFilename(request, mod);
  }

  require.resolve = resolve;

  require.main = process.mainModule;

  // Enable support to add extra extension types.
  require.extensions = Module._extensions;

  require.cache = Module._cache;

  return require;
}
```

各个模块文件中 ``require`` 的区别:
  * 内置模块（ ``module.js`` / ``fs.js`` 等 ）: 对应 ``NativeModule.require`` 函数，仅供 node 内部使用
  * 第三方模块: 对应 ``internalModule.makeRequireFunction()`` 函数的执行结果，底层依赖  ``Module.prototype.require()``，遵循 CommonJS 规范

## 总结

当执行 ``const xxx = require('xxx')`` 这段代码时
  1. 先根据 ``'xxx'`` 和模块所在目录得出被 require 的模块可能存在的目录 - [Module._resolveLookupPaths](#module_resolvelookuppaths)
  2. 再根据 ``'xxx'`` 和 1 的结果得出被 require 的模块的文件路径 - [Module._findPath](#module_findpath)
  3. 然后根据其拓展名确定加载方式 - [Module.prototype.load](#moduleprototypeload)
  4. 最后将 ``module.exports`` 导出

参考：
  * [https://github.com/nodejs/node/blob/master/lib/module.js](https://github.com/nodejs/node/blob/master/lib/module.js)
  * [https://github.com/nodejs/node/blob/master/lib/internal/module.js](https://github.com/nodejs/node/blob/master/lib/internal/module.js)