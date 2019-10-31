# 协程锁

增加协程锁组件，LocationComponent跟Actor都使用协程锁来实现队列机制，代码大大简化，并且非常好懂。
协程锁原理很简单，同一个Key只有一个协程能执行，其它同一个key的协程将队列，这个协程执行完会唤醒下一个协程。
协程锁是个非常方便的组件，比如服务端在处理登录或者下线过程中，每个异步操作都可能同一个账号会再次登录上来，
逻辑十分复杂，我们会希望登录某部分异步操作是原子操作，账号再次登录要等这个原子操作完成才能执行，
这样登录或者下线过程逻辑复杂度将会简化十倍以上。协程锁是ET解决这类异步重入问题的大杀器。



# CoroutineLock

```csharp
// 用协程锁组件创建一个Tcs任务,返回一个CoroutineLock
using (await CoroutineLockComponent.Instance.Wait(CoroutineLockType.ActorLocationSender, self.Id))
{
    // 返回这个Entity.Id对应的InstanceId
    self.ActorId = await Game.Scene.GetComponent<LocationProxyComponent>().Get(self.Id);
}
```

分解上述步骤:

1. 不同的协程锁类型都有一个`CoroutineLockQueueType`对象
  - `CoroutineLockQueueType`管理着`Dictionary<long, CoroutineLockQueue>`.
    - `CoroutineLockQueue`管理协程任务的`ETTaskCompletionSource<CoroutineLock>`队列queue.

```csharp
// 不同的coroutineLockType都有一个CoroutineLockQueueType
CoroutineLockQueueType coroutineLockQueueType = this.list[(int) coroutineLockType];
```

2. 如果还没有相同key的协程锁队列的话, 创建一个`CoroutineLockQueue`, 直接返回一个已完成的`ETTask<CoroutineLock>`

```csharp
// 如果字典中没有对应key的CoroutineLockQueue队列, 就创建一个
if (!coroutineLockQueueType.TryGetValue(key, out CoroutineLockQueue queue))
{
    queue = EntityFactory.Create<CoroutineLockQueue>(this.Domain);
    coroutineLockQueueType.Add(key, queue);
    // 用来创建一个带返回值的、已完成的ETTask<CoroutineLock>, 返回的是CoroutineLock。
    return ETTask.FromResult(EntityFactory.CreateWithParent<CoroutineLock, CoroutineLockType, long>(this, coroutineLockType, key));
}
```

3. 如果存在相同key的协程锁队列, 则创建一个`ETTaskCompletionSource<CoroutineLock> tcs`放入队列, 返回`tcs.Task`

4. 第一个拿到协程锁的协程, 直接拿到一个`using(CoroutineLock)`, 执行一个协程处理完成后, 会自动`Dipose`这个`CoroutineLock`, 协程锁`Dispose`方法会调用`Notify`
  - 判断对应`key`的`coroutineLockQueueType`队列中是否还有元素(`ETTaskCompletionSource<CoroutineLock> tcs`)
  - 没有元素就删除这个queue,并Dispose.
  - 如果有元素, 就取出队首的元素, 设置它的`tcs.SetResult`, 返回给它`CoroutineLock`, 在它用完后`Dispose`时继续调用`Notify`.

```csharp
// CoroutineLock的Dispose方法. 相应的, 需要用using来包围代码,或者手动Dispose释放协程锁
public override void Dispose()
{
    if (this.IsDisposed)
    {
        return;
    }
    base.Dispose();

    CoroutineLockComponent.Instance.Notify(coroutineLockType, this.key);
}
```




**这就保证了同一个Key只有一个协程能执行，其它同一个key的协程将队列，这个协程执行完会唤醒下一个协程。**


协程锁组件的一些代码注释:

```csharp
/// <summary>
/// 去获取协程锁来执行协程
/// </summary>
/// <param name="coroutineLockType"></param>
/// <param name="key"></param>
/// <returns></returns>
public ETTask<CoroutineLock> Wait(CoroutineLockType coroutineLockType, long key)
{
    // 取出对应CoroutineLockType的CoroutineLockQueueType类型(管理一个字典)
    CoroutineLockQueueType coroutineLockQueueType = this.list[(int) coroutineLockType];
    // 如果字典中没有对应key的CoroutineLockQueue队列, 就创建一个
    if (!coroutineLockQueueType.TryGetValue(key, out CoroutineLockQueue queue))
    {
        queue = EntityFactory.Create<CoroutineLockQueue>(this.Domain);
        coroutineLockQueueType.Add(key, queue);
        // 用来创建一个带返回值的、已完成的ETTask<CoroutineLock>, 返回的是CoroutineLock。
        return ETTask.FromResult(EntityFactory.CreateWithParent<CoroutineLock, CoroutineLockType, long>(this, coroutineLockType, key));
    }
    // 排队放入队列中,等待上一个CoroutineLock Dispose时调用Notify来唤醒队列中的下一个任务
    ETTaskCompletionSource<CoroutineLock> tcs = new ETTaskCompletionSource<CoroutineLock>();
    queue.Enqueue(tcs);
    return tcs.Task;
}

/// <summary>
/// 唤醒下一个需要协程锁的任务
/// </summary>
/// <param name="coroutineLockType"></param>
/// <param name="key"></param>
/// <exception cref="Exception"></exception>
public void Notify(CoroutineLockType coroutineLockType, long key)
{
    // 取出对应CoroutineLockType的CoroutineLockQueueType类型(管理一个字典)
    CoroutineLockQueueType coroutineLockQueueType = this.list[(int) coroutineLockType];
    // 如果字典中没有对应key的CoroutineLockQueue队列, 就创建一个
    if (!coroutineLockQueueType.TryGetValue(key, out CoroutineLockQueue queue))
    {
        throw new Exception($"first work notify not find queue");
    }
    if (queue.Count == 0)
    {
        coroutineLockQueueType.Remove(key);
        queue.Dispose();
        return;
    }

    ETTaskCompletionSource<CoroutineLock> tcs = queue.Dequeue();
    tcs.SetResult(EntityFactory.CreateWithParent<CoroutineLock, CoroutineLockType, long>(this, coroutineLockType, key));
}
```




# 异步重入

在应用程序中包含异步代码时，应该考虑并尽可能防止重入，即在异步操作完成之前重新输入。

https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/handling-reentrancy-in-async-apps

## 认识重入

例子: 用户点击按钮启动一个异步应用程序，该程序下载一系列网站并计算下载的总字节数。但是，如果用户多次点击按钮，则重复调用事件处理程序，并且每次都重新输入下载过程。结果，几个异步操作同时运行，输出交错了结果.

## 处理重入

1. 在操作运行时禁用Start按钮，这样用户就不能中断它。
  - 当操作正在运行时，可以通过禁用StartButton_Click事件处理程序顶部的按钮来阻止Start按钮。当操作完成时，您可以在finally块中重新启用按钮，以便用户可以再次运行应用程序。

  ```csharp
  private async void StartButton_Click(object sender, RoutedEventArgs e)
{
    // This line is commented out to make the results clearer in the output.
    //ResultsTextBox.Text = "";

    // ***Disable the Start button until the downloads are complete.
    StartButton.IsEnabled = false;

    try
    {
        await AccessTheWebAsync();
    }
    catch (Exception)
    {
        ResultsTextBox.Text += "\r\nDownloads failed.";
    }
    // ***Enable the Start button in case you want to run the program again.
    finally
    {
        StartButton.IsEnabled = true;
    }
}
  ```



2. 当用户再次选择Start按钮时，取消任何仍在运行的操作，然后让最近请求的操作继续。


```csharp
// 1. 声明一个CancellationTokenSource变量cts，它在所有方法的作用域内。
public partial class MainWindow : Window   // Or class MainPage
{
    // *** Declare a System.Threading.CancellationTokenSource.
    CancellationTokenSource cts;
    ...
}

// 2. 在StartButton_Click中，确定操作是否已经开始。如果cts的值为null，
// 则没有操作已经处于活动状态。如果该值不为空，则已经运行的操作将被取消。
if (cts != null)
{
  cts.Cancel();
}

// 3. 将cts设置为表示当前进程的不同值。
// 现在将cts设置为一个新值，您可以使用它来取消当前进程
CancellationTokenSource newCTS = new CancellationTokenSource();
cts = newCTS;

// 4. 在StartButton_Click结束时，当前进程已经完成，因此将cts的值设置回null。
if (cts == newCTS)
    cts = null;

```


3. 允许所有被请求的操作异步运行，但是要协调输出的显示，这样每个操作的结果就会一起按顺序显示。


```csharp
// *** Provide a parameter for the CancellationToken from StartButton_Click.
async Task AccessTheWebAsync(CancellationToken ct)
{
    // Declare an HttpClient object.
    HttpClient client = new HttpClient();

    // Make a list of web addresses.
    List<string> urlList = SetUpURLList();

    var total = 0;
    var position = 0;

    foreach (var url in urlList)
    {
        // *** Use the HttpClient.GetAsync method because it accepts a
        // cancellation token.
        HttpResponseMessage response = await client.GetAsync(url, ct);

        // *** Retrieve the website contents from the HttpResponseMessage.
        byte[] urlContents = await response.Content.ReadAsByteArrayAsync();

        // *** Check for cancellations before displaying information about the
        // latest site.
        ct.ThrowIfCancellationRequested();

        DisplayResults(url, urlContents, ++position);

        // Update the total.
        total += urlContents.Length;
    }

    // Display the total count for all of the websites.
    ResultsTextBox.Text +=
        $"\r\n\r\nTOTAL bytes returned:  {total}\r\n";
}

1. msdn.microsoft.com/library/hh191443.aspx                83732
2. msdn.microsoft.com/library/aa578028.aspx               205273
3. msdn.microsoft.com/library/jj155761.aspx                29019
4. msdn.microsoft.com/library/hh290140.aspx               122505
5. msdn.microsoft.com/library/hh524395.aspx                68959
6. msdn.microsoft.com/library/ms404677.aspx               197325
Download canceled.

1. msdn.microsoft.com/library/hh191443.aspx                83732
2. msdn.microsoft.com/library/aa578028.aspx               205273
3. msdn.microsoft.com/library/jj155761.aspx                29019
Download canceled.

1. msdn.microsoft.com/library/hh191443.aspx                83732
2. msdn.microsoft.com/library/aa578028.aspx               205273
3. msdn.microsoft.com/library/jj155761.aspx                29019
4. msdn.microsoft.com/library/hh290140.aspx               117152
5. msdn.microsoft.com/library/hh524395.aspx                68959
6. msdn.microsoft.com/library/ms404677.aspx               197325
7. msdn.microsoft.com                                            42972
8. msdn.microsoft.com/library/ff730837.aspx               146159

TOTAL bytes returned:  890591
```
