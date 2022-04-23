---
layout: post
title:  "async void导致程序崩溃"
categories: 编程
tags: csharp cs基础
excerpt: C#中要谨慎使用async void，因为它可能会导致程序崩溃。
---

* content
{:toc}

## 前言

之前都是在文档里看到：除了**winform的事件**可以使用`async void`，其他情况下**绝对不能**使用`async void`，而是要用`async Task`。

对于这个规范，虽然不是很明白内里原因，但是一直遵守着。

直到这天看到了这篇博客：[在 ASP.NET Core 中誤用 async void 竟引發了 502（Bad Gateway）](https://dotblogs.com.tw/supershowwei/2021/11/21/145303)，说`async void`里出现异常时会导致程序崩溃。研究测试了一番，终于明白原因。

摘录重点如下：
> 根據使用者提供的另一個線索「網站的某個功能壞了」，我們繼續往下追查，從程式碼當中我看到了一個近期新加的方法，它使用了 async void，沒錯，它使用了 async void，而且很不幸地它會發生 Exception，更慘的是這個 Exception 沒有被處理。
>
> 對 C# 非同步程式設計有了解的朋友，看到這邊應該大致上可以知道是發什麼問題了，[async void 是建議應該避免使用的宣告方式](https://docs.microsoft.com/zh-cn/archive/msdn-magazine/2013/march/async-await-best-practices-in-asynchronous-programming#避免-async-void)，其中一個原因就是當 async void 方法發生 Exception 時無法從呼叫端捕獲，即使加了 try...catch... 也沒用，async void 方法就有點像是我們自己起了另一個 Thread 去執行程式一樣，執行的過程中如果發生 Exception 沒有去處理，Exception 就會一路被往上拋，最終在 AppDomain 層級被捕獲，然後我們的應用程式就掛了。

## async-void-方法的异常无法被捕获

`async void`方法抛出的异常**无法被捕获**，异常会被一直往上面抛，最终在AppDomain层级被捕获，然后程序就挂了。

示例代码如下：
```cs
[HttpGet]
public async void Get()
{
    try
    {
        ThrowExceptionAsync();
    }
    catch (Exception ex)
    {
        //这里不能捕获到异常，程序崩溃！
        _logger.LogInformation(ex, "ex...");
    }
}

async void ThrowExceptionAsync()
{
    throw new Exception("async void ex!");
}
```

### 注意

前面所说的是 **`async void`方法抛出的无法预知到的异常**。在`async void`方法内部，我们仍然能够使用`try catch`，逻辑是正常逻辑。具体见下面的示例：
```cs
[HttpGet]
public async void Get()
{
    //在async void方法内部，我们仍然能够使用try catch，逻辑是正常逻辑。
    //此处try catch是有效的。异常被捕获处理了，async void方法执行无异常，不会导致程序崩溃。
    try
    {
        await Task.Run(() =>
        {
            throw new Exception("ex!");
        });
    }
    catch (Exception ex)
    {
        _logger.LogInformation(ex,"ex...");
    }
}
```

## 测试

### 崩溃

出现异常时能导致崩溃的代码有2种，如下：
```cs
[HttpGet]
public async void Get()
{
    //异常会导致程序崩溃
    throw new Exception("ex!");
}

[HttpGet]
public async void Get()
{
    //异常会导致程序崩溃
    await Task.Run(() =>
    {
        throw new Exception("ex!");
    });
}
```

#### 注意

下面的`async void`代码不会抛异常。
```cs
[HttpGet]
public async void Get()
{
    Task.Run(() =>
    {
        throw new Exception("ex!");
    });
}
```

代码里的`async void`没问题(不抛异常)，其实也符合逻辑。因为`async void`里面**没有异常**，自然就不会导致程序崩溃。

异常在`Task.Run`里面，因为**没有使用`await`进行等待**，那么异常就是被**线程池线程**捕获的，它们捕获到后，不会再往上面抛了，直接自己内部消化掉了。

因为`async void`在执行时**没有异常**，自然就不会导致程序崩溃。

但是由于我们不能保证所有代码都没有异常，所以**不要**使用`async void`！

### 不崩溃

只要**不是**`async void`，就算请求处理程序抛出了异常，也不会影响到**主线程**的。最多就是这次请求出错，返回`500 Internal Server Error`而已。

测试的几种代码如下：
```cs
[HttpGet]
public async Task Get()
{
    //500错误码
    throw new Exception("ex!");
}

[HttpGet]
public async Task Get()
{
    //500错误码
    await Task.Run(() =>
    {
        throw new Exception("ex!");
    });
}
```