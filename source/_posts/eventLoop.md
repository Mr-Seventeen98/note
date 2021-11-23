---
title: JavaScript 中的事件循环机制
date: 2021-10-12 22:08:12
tags:
---

JavaScript 是单线程的，一次只能处理一个任务。在正常的环境下这没有什么问题，但是试想一下你正在运行一个需要30秒的任务，在该任务期间我们需要等待30秒才能做其他事情（ JavaScript 默认是浏览器的主线程运行的，所以整个UI都会卡住，无响应）

幸运的是，浏览器为我们提供了一些 JavaScript 引擎本身不提供的功能：Web API。这包括 DOM API、setTimeout、HTTP 请求等。这可以帮助我们创建一些异步的、非阻塞的行为。

当我们调用一个函数时，它会被添加到称为执行栈中。执行栈是 JS 引擎的一部分，这不是特定于浏览器的。它是一个堆栈，意思是先入后出。当一个函数返回一个值时，它会出栈。

![gif](http://jsnext.icu/gid1%20%281%29.gif)

respond函数返回一个setTimeout函数。在setTimeout由Web API提供给我们：它让我们的任务延迟执行，而不会阻塞主线程。我们传递给setTimeout函数的是一个回调函数，`() => { return 'Hey'}`被添加到 Web API 中。与此同时，setTimeout函数和响应函数从堆栈中弹出，它们都返回了它们的值。

![gif](http://jsnext.icu/2.1.gif)

在 Web API 中，计时器运行的时间与我们传递给它的第二个参数一样长，即 1000 毫秒。回调不会立即添加到调用堆栈中，而是传递给事件队列。

![gif](http://jsnext.icu/3.1.gif)

回调函数并不是在 1000 毫秒后被添加到执行栈中（从而返回一个值）。它只是在 1000 毫秒后添加到事件队列中。函数的执行必须排队！这可能是令人疑惑的部分

是时候让事件循环完成它唯一的任务：将事件队列与执行栈连接起来！如果执行栈为空，那么如果所有先前调用的函数都返回了它们的值并出栈，则队事件列中的第一项将添加到执行栈中。在这种情况下，没有调用其他函数，这意味着当回调函数成为队列中的第一项时，执行栈是空的。

![gif](http://jsnext.icu/4.1.gif)

回调被添加到执行栈中，被调用，并返回一个值，然后出栈。

![gif](http://jsnext.icu/5.1.gif)

执行以下代码，想想整个处理过程是怎样的

```javascript
const foo = () => console.log("First");
const bar = () => setTimeout(() => console.log("Second"), 500);
const baz = () => console.log("Third");

bar();
foo();
baz();
```

让我们快速看看在浏览器中运行这段代码时发生了什么：

![gif](http://jsnext.icu/14.1.gif)

- 调用bar。 bar返回一个setTimeout函数。
- 我们传递给的回调函数被添加到 Web API、setTimeout函数并bar从执行栈中弹出。
- 计时器运行，同时foo被调用和输出`First`。foo返回（`undefined`），baz被调用，回调函数被添加到事件队列中。
- baz输出`Third`。事件循环在baz返回后看到执行栈为空，之后回调函数被添加到执行栈中。
- 回调函数被执行，输出`Second`。
