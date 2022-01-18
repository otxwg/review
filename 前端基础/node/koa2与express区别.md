# 概念

```
express 是一个基于Node.JSp平台的极简、灵活...，主要基于connenct中间件，自身封装了路由，视图处理。
koa相对更年轻，主要基于co中间件，基于es6 generator特性的异步流程控制，解决了回调地狱和麻烦的错误处理。
koa2是koa的2.0版本，使用assync和await实现异步流程控制
```

# 区别

```
1 express 自身集成了路由、视图，koa本身不继承任何中间件。
2 express 错误优先和callback koa2采用async await
3 错误处理 koa2 try catch
4 中间件模型 express 线性模型 koa2洋葱模型
5 context ：和express的只有request和response两个对象不同，koa增加了一个context对象，作为请求的上下文对象。同时，挂载了有request和response两个对象。提供了一些便捷方法。

```