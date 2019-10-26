> 要求熟读并背诵, 滑稽.jpg

# 基础

- CLR书: 27章 计算限制的异步处理
  - CLR线程池基础
  - 执行简单的计算限制操作
  - 执行上下文
  - 协作式取消和超时
  - 任务
    - 等待任务完成并获取结果
    - 取消任务
    - 任务完成时自动启动新任务
    - 任务可以启动子任务
    - 任务内部揭秘
    - 任务工厂
    - 任务调度器
    - ...
- CLR书: 28章 I-O限制的异步操作
  - Windows如何执行I/O操作
  - C#的异步函数
  - 编译器如何将异步函数转换成状态机
  - 异步函数转换成状态机详细流程
  - 执行上下文（Execution Context）
  - 异步函数的扩展性




# 什么是异步? 什么是协程?

[[ETBook]CSharp的协程](https://github.com/egametang/ET/blob/master/Book/2.1CSharp%E7%9A%84%E5%8D%8F%E7%A8%8B.md)

`OneThreadSynchronizationContext`是一个跨线程队列，任何线程可以往里面扔委托，`OneThreadSynchronizationContext`的`Update`方法在主线程中调用，会将这些委托取出来放到主线程执行。为什么回调方法需要扔回到主线程执行呢？因为回调方法中读取了loopCount，loopCount在主线程中也有读写，所以要么加锁，要么永远保证只在主线程中读写。加锁是个不好的做法，代码中到处是锁会导致阅读跟维护困难，很容易产生多线程bug。这种将逻辑打包成委托然后扔回另外一个线程是多线程开发中常用的技巧。

> 关于[同步上下文](/Async/同步上下文.md)

... 这里可以回答什么是协程了，其实这一串串回调就是协程。

## 单线程异步

[[ETBook]单线程异步](https://github.com/egametang/ET/blob/master/Book/2.3%E5%8D%95%E7%BA%BF%E7%A8%8B%E5%BC%82%E6%AD%A5.md)

因此await并不会开启多线程，await具体用没用多线程完全取决与具体的实现.

# 更好的协程

[[ETBook]更好的协程](https://github.com/egametang/ET/blob/master/Book/2.2%E6%9B%B4%E5%A5%BD%E7%9A%84%E5%8D%8F%E7%A8%8B.md)

... 还有个技巧，我们发现`WaitTime`中需要将`tcs.SetResult`扔回到主线程执行, 在`WaitTime`中直接调用`tcs.SetResult(true)`就行了，回调会自动扔到同步上下文中，而同步上下文我们可以在主线程中取出回调执行，这样自动能够完成回到主线程的操作.

如果不设置`同步上下文`，你会发现打印出来当前线程就不是主线程了，这也是很多第三方库跟net core内置库的用法，默认不回调到主线程，所以我们使用的时候需要设置下同步上下文。其实这个设计本人觉得没有必要，交由库的开发者去实现更好，尤其是在游戏开发中，逻辑全部是单线程的，回调每次都走一遍同步上下文就显得多余了，所以ET框架提供了不使用同步上下文的实现`ETTask`，代码更加简洁更加高效


# 到底ETTask是什么东东? 为什么不用Task?


1. ETTask只能用unity2018.3以上版本
2. ETTask是一个轻量级单线程的task，相比Task性能更强，GC更少
3. `async ETVoid`可以不需要`try` `catch` (`async void`中需要自己去`try` `catch`)
4. 取消任务`public ETTask<IResponse> Call(IRequest request, CancellationToken cancellationToken)`

---

【群主】熊猫(80081771)
永远使用`ETTask`就行了，绝对不用`Task`
【群主】熊猫(80081771)
`Task`里面很多方法是**多线程**实现的，没搞清楚随便用会出问题。`ETtask`完全是**单线程**的，不会提供多线程的方法，你想用也没法用。这好比lua。lua虽然没有多线程，但是能够保证新手都不容易出错.
【群主】熊猫(80081771)
ET 中`ETVoid`跟`async void`,一样的用法.

---



【群主】熊猫(80081771)
`ETVoid`替代`async void`，性能大大提高, 不用`ETVoid`就要用`async void`，`async void`生成的东西比较复杂，性能稍微差一点.

> 注意master中不要使用async void，要用ETVoid代替，这样才能捕获异常打印log，否则asyncvoid中需要自己去try catch

---

【群主】熊猫(80081771)
把ETTask当成Task使用就行了

【群主】熊猫(80081771)
这两个使用很简单啊，`etvoid`是代替`async void`，**意思是新开一个协程**。`ETTask`跟`Task`一样。当然`Task`不去`await`也相当于新开协程，但是编辑器会冒出提示，提示你`await`。所以新开协程最好用`ETVoid`。4.0用asyncvoid。使用场景，自己写写就明白啦

【群主】熊猫(80081771)
**协程就是回调** 啊

> `Task`用于没有结果的异步方法（即过程），而`Task<T>`用于返回结果的异步方法（即函数）。剩下的async void方法主要是为了与事件处理程序（`void Button_Click(object sender, EventArgs e)`那种）向后兼容而存在的，应避免在其他地方使用，因为它们内部未处理的异常将使整个过程崩溃。

# 关于异步的几篇文章

【码神】烟雨迷离半世殇(1778139321)
[解析C#中的异步方法](https://www.lfzxb.top/dissecting-the-async-methods-in-c/)
[拓展C#中的异步方法](https://www.lfzxb.top/extending-the-async-methods-in-c/)
[C#中异步方法的特征性能特征](https://www.lfzxb.top/the-performance-characteristics-of-async-methods-in-c/)

---

[[翻译]剖析C#中的异步方法 raytheweak ](https://www.cnblogs.com/raytheweak/p/8735141.html)
[[翻译]扩展C#中的异步方法 raytheweak ](https://www.cnblogs.com/raytheweak/p/9130594.html)
[[翻译]C#中异步方法的性能特点 raytheweak ](https://www.cnblogs.com/raytheweak/p/9314229.html)
[[翻译]用一个用户场景来掌握它们 raytheweak ](https://www.cnblogs.com/raytheweak/p/9383273.html)

---


[官方文档“类任务”（Task-like）类型](https://github.com/dotnet/roslyn/blob/master/docs/features/task-types.md)
[一些参考代码](https://github.com/SergeyTeplyakov/EduAsync/tree/master/src)


---

[关于AsyncMethodBuilder](http://blog.i3arnon.com/2016/07/25/arbitrary-async-returns/)
