---
layout: post
title: code-server相关学习 
date:   2021-4-8 8:58:30
category: "vscode"
---

## node.js express模块的介绍与简单使用
https://blog.csdn.net/weixin_45060872/article/details/107801864


## compression
gzip 压缩

## NodeJS的Promise的用法
　Javascript的特点是异步，Javascript不能等待，如果你实现某件需要等待的事情，你不能停在那里一直等待结果回来，相反，底线是使用回调callback：你定义一个函数，这个函数只有等到结果可用时才能被调用。

　这种回调模型对于好的代码组织是没有问题的，但是也可以通过从原始回调切换到promise解决很多问题，将promise看成是一个标准的数据容器，这样会简化你的代码组织，可以成为基于promise的架构。

### 什么是Promise?
　一个promise是一个带有".then()"方法的对象，其代表的是一个操作的结果可能还没有或不知道，无论谁访问这个对象，都能够使用".then()"方法加入回调等待操作出现成功结果或失败时的提醒通知，。

　那么为什么这样做好处优于回调呢？标准的回调模式在我们处理请求时需要同时提供回调函数：

```
request(url, function(error, response) {

  // handle success or error.

});

doSomethingElse();

```
　很不幸，这段代码意味着这个request函数并不知道它自己什么时候能够完成，当然也没有必要，我们最终通过回调传递结果。这会导致多个回调形成了嵌套回调，或者称为回调陷阱。

```
queryTheDatabase(query, function(error, result) {

  request(url, function(error, response) {

    doSomethingElse(response, function(error, result) {

      doAnotherThing(result, function(error, result) {

        request(anotherUrl, function(error, response) {

          ...

        });

      });

    });

  });

});
```

　Promise能够解决这种问题，允许低层代码创建一个request然后返回一个对象，其代表着未完成的操作，让调用者去决定应该加入什么回调。

### Promise是什么？
　promise是一个异步编程的抽象，它是一个返回值或抛出exception的代理对象，一般promise对象都有一个then方法，这个then方法是我们如何获得返回值(成功实现承诺的结果值，称为fulfillment)或抛出exception(拒绝承诺的理由，称为rejection)，then是用两个可选的回调作为参数，我们可以称为onFulfilled和OnRejected：
```
var promise = doSomethingAync()
promise.then(onFulfilled, onRejected)
```
　当这个promise被解决了，也就是异步过程完成后，onFulfilled和OnRejected中任何一个将被调用，

　因此，一个promise有下面三个不同状态：

- pending待承诺 - promise初始状态
- fulfilled实现承诺 - 一个承诺成功实现状态
- rejected拒绝承诺 - 一个承诺失败的状态

　以读取文件为案例，下面是使用回调实现读取文件后应该做什么事情(输出打印)：

```
readFile(function (err, data) {

  if (err) return console.error(err)

  console.log(data)

})
```
　如果我们的readFile函数返回一个promise，那么我们可以如下实现同样的逻辑(输出打印)：

```
var promise = readFile()
promise.then(console.log, console.error)
```
　这里我们有了一个值promise代表的是异步操作，我们能够一直传递这个值promise，任何人访问这个值都能够使用then来消费使用它，无论这个值代表的异步操作是否完成或没有完成，我们也能保证异步的结果不会改变，因为这个promise代表的异步操作只会执行一次，状态是要么fulfilled要么是rejected。

#### [更详细内容](https://www.cnblogs.com/linwenbin/p/12656664.html)





