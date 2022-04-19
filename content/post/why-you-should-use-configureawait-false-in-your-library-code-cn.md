---
title: 翻译：为什么应该在类库里使用ConfigureAwait(false)
date: 2017-12-11 13:04:09
tags: [C#, TPL]
---

**源文链接**：[C#: Why you should use ConfigureAwait(false) in your library code](https://medium.com/bynder-tech/c-why-you-should-use-configureawait-false-in-your-library-code-d7837dce3d7f) （翻译：[woodylic](https://woodylic.github.io)）

自从.NET 4.5引入了async/await以后，编写异步代码变得相当容易。Async/await关键字使得异步代码看起来和同步代码非常类似，这大大提高了代码的可读性以及编程效率，这多亏了编译器处理了异步编程最困难的部分。

<!--more-->

下面例子展示了如何通过简单的代码完成一个异步操作：访问一个特定url，并把response内容写入字符串：

```csharp
public async Task<string> DoCurlAsync()
{
    using (var httpClient = new HttpClient())
    using (var httpResonse = await httpClient.GetAsync("https://www.bynder.com"))
    {
        return await httpResonse.Content.ReadAsStringAsync();
    }
}
```

至此，我们就拥有了一个从bynder.com获取内容的异步调用。理想情况下，这个异步调用会按以下方式被调用：

```csharp
var bynderContents = await DoCurlAsync();
```

但实际上，有些开发人员可能会按以下方式调用：

```csharp
var bynderContents = DoCurlAsync().Result;
```

在这种情况下，curl将以同步方式运行，调用进程会被阻塞至curl结束为止。如果这段代码在Console应用里执行，大部分时候也能正常执行。

但是，如果代码在一个UI应用执行，例如在一个按钮点击事件中触发：

```csharp
public void OnButtonClicked(object sender, RoutedEventArgs e)
{
    var bynderContents = DoCurlAsync().Result;
}
```

应用将会因为死锁而僵死，类库的用户也会抱怨我们的类库导致了应用无响应。

为了解决这个问题，我们可以重写我们的异步调用：

```csharp
public async Task<string> DoCurlAsync()
{
    using (var httpClient = new HttpClient())
    using (var httpResponse = await httpClient.GetAsync("https://www.bynder.com").ConfigureAwait(false))
    {
        return await httpResponse.Content.ReadAsStringAsync().ConfigureAwait(false);
    }
}
```

实际上，只要为httpClient.GetAsync()加上ConfigureAwait(false)就可以解决问题了。

```csharp
public async Task<string> DoCurlAsync()
{
    using (var httpClient = new HttpClient())
    using (var httpResonse = await httpClient.GetAsync("https://www.bynder.com").ConfigureAwait(false))
    {
        return await httpResonse.Content.ReadAsStringAsync();
    }
}
```

总的来说，总是为你的类库的异步调用加上ConfigureAwait(false)是一种推荐的做法，这可以避免（调用方造成的）不必要的问题。

接下来，我们会分析为什么（不当的async调用）在UI应用（以及少数Console应用）会造成死锁，以及为什么ConfigureAwait(false)可以解决死锁问题。

首先我们要了解UI应用的工作原理：

- 只有一个进程，即UI进程，负责UI渲染工作。任何UI变更都只能通过UI进程调用，所以一旦UI进程被阻塞，应用就会变为无响应状态。

- UI进程有一个消息队列，用以接收将要执行的通知和动作。在Win32的实现如下：

```csharp
while((bRet = GetMessage( &msg, NULL, 0, 0 )) != 0)
{
    // No errors are handled, for simplicity purposes.
    TranslateMessage(&msg);
    DispatchMessage(&msg);
}
```

- UI线程默认都会有一个同步上下文（SynchronizationContext）。

在UI进程执行了异步调用后，await后的代码会回到UI进程的SynchronizationContext，这是默认的并且是期望的行为。

如果从一个非UI进程修改一个UI组件，会抛出System.InvalidOperationException。

```csharp
public void OnButtonClicked(object sender, RoutedEventArgs e)
{
    var bynderContents = await DoCurlAsync();
    myTextBlock.Text = bynderContents;
}
```

回到上面例子，DoCurlAsync从概念上等同于如下代码：

```csharp
var currentContext = SynchronizationContext.Current;
var httpResponseTask = httpClient.GetAsync("https://www.bynder.com");
httpResponseTask.ContinueWith(delegate
{
    if (currentContext == null)
    {
        return await httpResonse.Content.ReadAsStringAsync();
    }
    else
    {
        currentContext.Post(delegate {
            await httpResonse.Content.ReadAsStringAsync();
        }, null);
     }
}, TaskScheduler.Current);
```

注意：这里的代码片段只是await实际编译出来的代码的简化版本，而且也没有使用using来处理资源的关闭和释放。

Post调用会向UI进程的消息泵发送一条待处理消息，所以为了让DoCurlAsync调用完成，必须在UI进程执行await httpResonse.Content.ReadAsStringAsync()。

但是，在下面这个例子中：

```csharp
public void OnButtonClicked(object sender, RoutedEventArgs e)
{
    var bynderContents = DoCurlAsync().Result;
}
```

UI进程无法执行await httpResonse.Content.ReadAsStringAsync()，因为它已经被DoCurlAsync().Result阻塞了。这就产生了一个死锁，DoCurlAsync将无法结束。ConfigureAwait(false)配置指定await后的操作无需回到原来的context，因此解决了可能的异常。

References:

[Asynchronous Programming with async and await (C#)](https://docs.microsoft.com/en-us/dotnet/csharp/async)

[Await, SynchronizationContext, and Console Apps](https://blogs.msdn.microsoft.com/pfxteam/2012/01/20/await-synchronizationcontext-and-console-apps/)

[Parallel Programming with .NET](https://blogs.msdn.microsoft.com/pfxteam/)
