+++
title = "详解 KOA 洋葱模型的实现原理" 
date = 2024-11-11
draft = false 
tags = ["KOA"] 
+++

## 背景

面试的时候面试官问我知道 KOA 的洋葱模型是怎么实现的吗？
一下把我问住了，我确实没有研究过是如何实现的。自己头脑风暴了一会也没猜到作者的思路。

面试结束以后，赶紧研究了一下源码，看看框架作者是怎么写的。
果然是大神！利用递归的特性轻轻松松几行代码就实现了。

于是就写了这篇文章拆解一下作者的实现思路。

## 前置知识

在看源码之前需要先了解一下这两个概念

- 洋葱模型
- 递归
<!--more-->

### 什么是洋葱模型

洋葱模型是 Koa 框架的核心思想之一，用于描述中间件的执行顺序。

中间件的执行顺序类似于洋葱的层次结构，从外到内，再从内到外。

每个中间件可以在请求处理的不同阶段执行代码。

```js
const Koa = require("koa");
const app = new Koa();

app.use(async (ctx, next) => {
  console.log("中间件1 - 开始");
  await next();
  console.log("中间件1 - 结束");
});

app.use(async (ctx, next) => {
  console.log("中间件2 - 开始");
  await next();
  console.log("中间件2 - 结束");
});

app.use(async (ctx, next) => {
  console.log("中间件3 - 开始");
  ctx.body = "Hello Koa";
  console.log("中间件3 - 结束");
});

app.listen(3000, () => {
  console.log("服务器启动在 http://localhost:3000");
});

//执行结果
当请求到达服务器时，执行顺序如下：

中间件1 - 开始
中间件2 - 开始
中间件3 - 开始
中间件3 - 结束
中间件2 - 结束
中间件1 - 结束
```

### 什么是递归呢？

递归是一种在函数定义中使用函数自身的方法。

递归函数会重复调用自身，直到满足某个条件为止。

```js
const factorial = (n) => {
  // 终止条件
  if (n === 0) {
    return 1;
  }
  // 调用自身
  return n * factorial(n - 1);
};

console.log(factorial(5)); // 输出 120
```

对于上面的调用，我们大概可以画出这么一个调用栈。

```js
┌───┐
│ f(0)
├───┤
│ f(1)
├───┤
│ f(2)
├───┤
│ f(3)
├───┤
│ f(4)
├───┤
│ f(5)
└───┘
```

当到达终止条件以后，调用栈会逐层返回，直到返回到最初的调用。

所以会先计算 f(0) 然后再计算 f(1) 以此类推:

```js
f(0) = 1
↓
f(1) = 1 * f(0) = 1
↓
f(2) = 2 * f(1) = 2
↓
f(3) = 3 * f(2) = 6
↓
f(4) = 4 * f(3) = 24
↓
f(5) = 5 * f(4) = 120
```

所以不难理解,递归先是一层一层的调用，然后再一层一层的返回。

```js
f(5) -> f(4) -> f(3) -> f(2) -> f(1) -> f(0)
f(0) -> f(1) -> f(2) -> f(3) -> f(4) -> f(5)
```

这么浅浅一看，是不是和上面的洋葱模型的描述有点像呢。
假如我想模拟一下上面的洋葱模型的执行顺序，就可以这么做：

```js
const factorialOnion = (n) => {
    if (n === 0) {
      console.log("=====")
      return 1;
    }
    console.log(`${n} - 开始`);
    const result = n * factorialOnion(n - 1);
    console.log(`${n} - 结束`);
    return result;
  };

factorialOnion(5)

//运行结果
  5 - 开始
  4 - 开始
  3 - 开始
  2 - 开始
  1 - 开始
  =====
  1 - 结束
  2 - 结束
  3 - 结束
  4 - 结束
  5 - 结束
```

## 拆解

理解了上米的两个概念，我们就可以来看看 KOA 的洋葱模型是怎么实现的了。

[先来看看源码](https://github.com/koajs/compose/blob/master/index.js)

```js
function compose(middleware) {
  if (!Array.isArray(middleware))
    throw new TypeError("Middleware stack must be an array!");
  for (const fn of middleware) {
    if (typeof fn !== "function")
      throw new TypeError("Middleware must be composed of functions!");
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1;
    return dispatch(0);
    function dispatch(i) {
      if (i <= index)
        return Promise.reject(new Error("next() called multiple times"));
      index = i;
      let fn = middleware[i];
      if (i === middleware.length) fn = next;
      if (!fn) return Promise.resolve();
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err);
      }
    }
  };
}
```

这一部分是类型检查，确保传入的中间件是一个数组，并且每个中间件都是一个函数。

```js
if (!Array.isArray(middleware))
  throw new TypeError("Middleware stack must be an array!");
for (const fn of middleware) {
  if (typeof fn !== "function")
    throw new TypeError("Middleware must be composed of functions!");
}
```

执行的核心是下面这部分。
可以看到返回的是一个函数，这个函数接收两个参数，一个是 context，一个是 next。

```js
return function (context, next) {
  // last called middleware #
  let index = -1;
  return dispatch(0);
  function dispatch(i) {
    if (i <= index)
      return Promise.reject(new Error("next() called multiple times"));
    index = i;
    let fn = middleware[i];
    if (i === middleware.length) fn = next;
    if (!fn) return Promise.resolve();
    try {
      return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
    } catch (err) {
      return Promise.reject(err);
    }
  }
};
```

接下来结合[源码的测试用例](https://github.com/koajs/compose/blob/master/test/test.js) 用例来解释一下，看下上面`compose`核心代码是怎么执行的吧。

```js
describe('Koa Compose', function () {
  it('should work', async () => {
    const arr = []
    const stack = []

    stack.push(async (context, next) => {
      arr.push(1)
      await wait(1)
      await next()
      await wait(1)
      arr.push(6)
    })

    stack.push(async (context, next) => {
      arr.push(2)
      await wait(1)
      await next()
      await wait(1)
      arr.push(5)
    })

    stack.push(async (context, next) => {
      arr.push(3)
      await wait(1)
      await next()
      await wait(1)
      arr.push(4)
    })

    await compose(stack)({})
    expect(arr).toEqual(expect.arrayContaining([1, 2, 3, 4, 5, 6]))
  })
```

首先执行 `dispatch(0)`

```js
return function (context, next) {
  let index = -1;
  return dispatch(0); <-- 这里
  ...
```

伪代码大概是这样:

```js
  function dispatch(0) {
    if (0 <= -1)
      return Promise.reject(new Error("next() called multiple times"));
    index = 0;
    let fn = middleware[0];
    if (0 === 3) fn = next;
    if (!fn) return Promise.resolve();
    try {
      return Promise.resolve(fn(context, dispatch.bind(null, 1))); <-- 进入这里
    } catch (err) {
      return Promise.reject(err);
    }
  }
```

通过执行 `return Promise.resolve(fn(context, dispatch.bind(null, 1)))` 这行代码
就会进入第一个中间件的执行

```js
arr.push(1);
await wait(1);
await next(); <- 进入这里
await wait(1);
arr.push(6);
```

当执行到 `await next()` 的时候，就会执行`dispatch.bind(null, 1)`这个函数，从而进入下二个中间件的执行。伪代码大概如下：

```js
  function dispatch(1) {
    if (1 < 0)
      return Promise.reject(new Error("next() called multiple times"));
    index = 1;
    let fn = middleware[1];
    if (1 === 3) fn = next;
    if (!fn) return Promise.resolve();
    try {
      return Promise.resolve(fn(context, dispatch.bind(null, 2))); <-- 进入这里
    } catch (err) {
      return Promise.reject(err);
    }
  }
```

然后执行第二个中间件

```js
arr.push(2);
await wait(1);
await next(); < --进入这里;
await wait(1);
arr.push(5);
```

循环往复，根据递归的特性，一层一层的调用，再一层一层地返回。
最后 arr 的内容就会是 `[1, 2, 3, 4, 5, 6]`。

## 总结

可以看到，大神通过递归的特性，一层一层的调用，再一层一层地返回，轻松实现了 KOA 的洋葱模型。

太妙了!
