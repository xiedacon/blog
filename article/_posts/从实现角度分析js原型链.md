---
title: 从实现角度分析js原型链
date: 2017-08-30
tags: js
---

# 从实现角度分析js原型链

网上介绍原型链的优质文章已经有很多了，比如说：
  * [https://github.com/mqyqingfeng/Blog/issues/2](https://github.com/mqyqingfeng/Blog/issues/2)
  * [https://github.com/creeperyang/blog/issues/9](https://github.com/creeperyang/blog/issues/9)

作为补充，就让我们换个角度，从实现来分析一下吧

> ps: 本文假设你对原型链已经有所了解。如不了解，建议先看上面两篇文章

## 画个图

### 第一步

创建一个函数时，会创建两个对象：函数本身和它的原型对象

![第一步](http://upload-images.jianshu.io/upload_images/2395997-92d14a46b28b1172.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以我们可以先画个这样的关系图：

![示例1](http://upload-images.jianshu.io/upload_images/2395997-63c51ccfd115d06c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> ps: 圆形代表函数，矩形代表对象

### 第二步

通过函数创建的对象，其原型是函数的原型对象

![第二步](http://upload-images.jianshu.io/upload_images/2395997-4d698c04cb6fca06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再修改下关系图：

![示例2](http://upload-images.jianshu.io/upload_images/2395997-656bafd57a409ecd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第三步

函数的原型对象的原型是 Object 的原型对象

![第三步](http://upload-images.jianshu.io/upload_images/2395997-b36e079999f66cc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再修改下关系图：

![示例3](http://upload-images.jianshu.io/upload_images/2395997-92dc447a25936439.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第四步

js的内置函数对象也满足这个规律

![第四步](http://upload-images.jianshu.io/upload_images/2395997-0dce0597870e3a14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再修改下关系图：

![示例4](http://upload-images.jianshu.io/upload_images/2395997-fe5a7d3f084db4aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第五步

Function 的原型对象是一个函数

![第五步](http://upload-images.jianshu.io/upload_images/2395997-24c24a9311003eb2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再修改下关系图：

![示例5](http://upload-images.jianshu.io/upload_images/2395997-5ab3b9d42967f14f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第六步

所有函数的原型都相同，都为 Function 的原型对象

![第六步](http://upload-images.jianshu.io/upload_images/2395997-e7e39bcfb0c0c04d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再修改下关系图：

![示例6](http://upload-images.jianshu.io/upload_images/2395997-71ebe44042767ef4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 第七步

Object 的原型对象的原型是 null 意为不应该存在

![第七步](http://upload-images.jianshu.io/upload_images/2395997-f32d7b4b32e14ff2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后得到如下关系图：

![关系图](http://upload-images.jianshu.io/upload_images/2395997-60c26f950a32003b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 一些疑问

### instanceof

```js
Object instanceof Function // true
Function instanceof Object // true
```

首先需要确定的是，instanceof 运算符相当于如下代码: 

```js
// L instanceof R
function instance_of(L, R) {
 var O = R.prototype; // 取函数 R 的原型对象
 L = L.__proto__;     // 取对象 L 的原型
 while (true) {       // 遍历原型链
   if (L === null)
     return false; 
   if (O === L)       // 函数 R 的原型对象在对象 L 的原型链上
     return true; 
   L = L.__proto__; 
 } 
}
```

对于 ``Object instanceof Function`` 来说，就相当于 ``Object.__proto__ === Function.prototype``

因为所有函数的原型都是 Function 的原型对象，所以是 true

对于 ``Function instanceof Object`` 来说，就相当于 ``Function.__proto__ === Object.prototype``

因为 Object 的原型对象处于原型链的顶部，所以 ``Object.prototype`` 一定在 Function 的原型链上，为 true

### Function

```js
Function.__proto__ === Function.prototype
```

对于这个，可以先把上面的关系图变形一下：

![变形](http://upload-images.jianshu.io/upload_images/2395997-55f4dc77fa68206a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出：
  1. 所有函数都有与之对应的原型对象
  2. 所有函数的原型都是 ``Function.prototype``
  3. ``Object.prototype`` 位于原型链的顶部，在所有对象的原型链上

根据 1 和 2，就可以得出 ``Function.__proto__ === Function.prototype``

至于 ``Function.prototype`` 为什么是函数，可以先看看下面这个例子：

![例子](http://upload-images.jianshu.io/upload_images/2395997-76230c482f59211c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看出，``Array.prototype`` 是 Array 类型，``Map.prototype`` 是 Map 类型，``Set.prototsype`` 是 Set 类型

所以，为了保持一致性，``Function.prototype`` 也应该是 Function 类型

## End

参考：
 * [https://github.com/mqyqingfeng/Blog/issues/2](https://github.com/mqyqingfeng/Blog/issues/2)
 * [https://github.com/creeperyang/blog/issues/9](https://github.com/creeperyang/blog/issues/9)
 * [https://www.ibm.com/developerworks/cn/web/1306_jiangjj_jsinstanceof/index.html](https://www.ibm.com/developerworks/cn/web/1306_jiangjj_jsinstanceof/index.html)
 * [https://stackoverflow.com/questions/7688902/what-is-functions-proto](https://stackoverflow.com/questions/7688902/what-is-functions-proto)
 * [https://stackoverflow.com/questions/572897/how-does-javascript-prototype-work](https://stackoverflow.com/questions/572897/how-does-javascript-prototype-work)