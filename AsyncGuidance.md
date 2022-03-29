# Table of contents
 - [Asynchronous Programming](#asynchronous-programming)
   - [Asynchrony is viral](#asynchrony-is-viral)
   - [Async void](#async-void)
   - [Prefer Task.FromResult over Task.Run for pre-computed or trivially computed data](#prefer-taskfromresult-over-taskrun-for-pre-computed-or-trivially-computed-data)
   - [Avoid using Task.Run for long running work that blocks the thread](#avoid-using-taskrun-for-long-running-work-that-blocks-the-thread)
   - [Avoid using Task.Result and Task.Wait](#avoid-using-taskresult-and-taskwait)
   - [Prefer await over ContinueWith](#prefer-await-over-continuewith)
   - [Always create TaskCompletionSource\<T\> with TaskCreationOptions.RunContinuationsAsynchronously](#always-create-taskcompletionsourcet-with-taskcreationoptionsruncontinuationsasynchronously)
   - [Always dispose CancellationTokenSource(s) used for timeouts](#always-dispose-cancellationtokensources-used-for-timeouts)
   - [Always flow CancellationToken(s) to APIs that take a CancellationToken](#always-flow-cancellationtokens-to-apis-that-take-a-cancellationtoken)
   - [Cancelling uncancellable operations](#cancelling-uncancellable-operations)
   - [Always call FlushAsync on StreamWriter(s) or Stream(s) before calling Dispose](#always-call-flushasync-on-streamwriters-or-streams-before-calling-dispose)
   - [Prefer async/await over directly returning Task](#prefer-asyncawait-over-directly-returning-task)
   - [ConfigureAwait](#configureawait)
 - [Scenarios](#scenarios)
   - [Timer callbacks](#timer-callbacks)
   - [Implicit async void delegates](#implicit-async-void-delegates)
   - [ConcurrentDictionary.GetOrAdd](#concurrentdictionarygetoradd)
   - [Constructors](#constructors)
   - [WindowsIdentity.RunImpersonated](#windowsidentityrunimpersonated)
 
# Asynchronous Programming

Asynchronous programming has been around for several years on the .NET platform but has historically been very difficult to do well. Since the introduction of async/await in C# 5 asynchronous programming has become mainstream. Modern frameworks (like ASP.NET Core) are fully asynchronous and it's very hard to avoid the async keyword when writing web services. As a result, there's been lots of confusion on the best practices for async and how to use it properly. This section will try to lay out some guidance with examples of bad and good patterns of how to write asynchronous code.

비동기 프로그래밍은 .NET 플랫폼에서 몇 년 동안 사용되었지만 역사적으로 잘 수행하기가 매우 어려웠습니다. C# 5에 async/await가 도입된 이후로 비동기 프로그래밍이 주류가 되었습니다. ASP.NET Core와 같은 최신 프레임워크는 완전히 비동기적이며 웹 서비스를 작성할 때 async 키워드를 피하기가 매우 어렵습니다. 결과적으로 비동기에 대한 모범 사례와 올바르게 사용하는 방법에 대해 많은 혼란이 있었습니다. 이 섹션에서는 비동기 코드를 작성하는 방법에 대한 나쁜 패턴과 좋은 패턴의 예와 함께 몇 가지 지침을 제시하려고 합니다.
## Asynchrony is viral 모든 호출 비동기화

Once you go async, all of your callers **SHOULD** be async, since efforts to be async amount to nothing unless the entire callstack is async. In many cases, being partially async can be worse than being entirely synchronous. Therefore it is best to go all in, and make everything async at once.

일단 비동기화되면 전체 호출 스택이 비동기화되지 않는 한 비동기화하려는 노력이 아무 소용이 없기 때문에 모든 호출자는 비동기화되어야 합니다 **SHOULD**.많은 경우에 부분적으로 비동기화하는 것이 완전히 동기화되는 것보다 나쁠 수 있습니다. 따라서 올인하고 모든 것을 한 번에 비동기화하는 것이 가장 좋습니다.


❌ **BAD** This example uses the `Task.Result` and as a result blocks the current thread to wait for the result. This is an example of [sync over async](#avoid-using-taskresult-and-taskwait).

❌ **BAD** 이 예제는 `Task.Result`를 사용하며 결과적으로 현재 스레드가 결과를 기다리도록 차단합니다. [sync over async](#avoid-using-taskresult-and-taskwait)의 예입니다. 36
```C#
public int DoSomethingAsync()
{
    var result = CallDependencyAsync().Result;
    return result + 1;
}
```

:white_check_mark: **GOOD** This example uses the await keyword to get the result from `CallDependencyAsync`.

:white_check_mark: **GOOD** 이 예제에서는 await 키워드를 사용하여 `CallDependencyAsync`에서 결과를 가져옵니다. 47


```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Async void 형식 금지

Use of async void in ASP.NET Core applications is **ALWAYS** bad. Avoid it, never do it. Typically, it's used when developers are trying to implement fire and forget patterns triggered by a controller action. Async void methods will crash the process if an exception is thrown. We'll look at more of the patterns that cause developers to do this in ASP.NET Core applications but here's a simple example:

ASP.NET Core 애플리케이션에서 async void를 사용하는 것은 **항상** 좋지 않습니다. 피하세요, 절대 하지 마세요. 일반적으로 개발자가 컨트롤러 작업에 의해 트리거된 화재 및 잊어버리기 패턴을 구현하려고 할 때 사용됩니다.비동기 void 메서드는 예외가 발생하면 프로세스를 중단시킵니다. 개발자가 ASP.NET Core 애플리케이션에서 이 작업을 수행하도록 하는 더 많은 패턴을 살펴보겠지만 다음은 간단한 예입니다.



❌ **BAD** Async void methods can't be tracked and therefore unhandled exceptions can result in application crashes.

 비동기 무효 메서드는 추적할 수 없으므로 처리되지 않은 예외로 인해 애플리케이션이 충돌할 수 있습니다. 68

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        BackgroundOperationAsync();
        return Accepted();
    }
    
    public async void BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

:white_check_mark: **GOOD** `Task`-returning methods are better since unhandled exceptions trigger the [`TaskScheduler.UnobservedTaskException`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception?view=netframework-4.7.2).

 `Task` - 처리되지 않은 예외가 [`TaskScheduler.UnobservedTaskException`]을 트리거하므로 메서드를 반환하는 것이 더 좋습니다.

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        Task.Run(BackgroundOperationAsync);
        return Accepted();
    }
    
    public async Task BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

## Prefer `Task.FromResult` over `Task.Run` for pre-computed or trivially computed data

미리 계산되거나 간단하게 계산된 데이터의 경우 'Task.Run'보다 'Task.FromResult'를 선호합니다.


For pre-computed results, there's no need to call `Task.Run`, that will end up queuing a work item to the thread pool that will immediately complete with the pre-computed value. Instead, use `Task.FromResult`, to create a task wrapping already computed data.

미리 계산된 결과의 경우 'Task.Run'을 호출할 필요가 없습니다. 그러면 작업 항목이 미리 계산된 값으로 즉시 완료되는 스레드 풀에 대기열에 추가됩니다. 대신 'Task.FromResult'를 사용하여 이미 계산된 데이터를 래핑하는 작업을 만듭니다.



❌ **BAD** This example wastes a thread-pool thread to return a trivially computed value.

❌ **BAD** 이 예제는 간단하게 계산된 값을 반환하기 위해 스레드 풀 스레드를 낭비합니다. 123


```C#
public class MyLibrary
{
   public Task<int> AddAsync(int a, int b)
   {
       return Task.Run(() => a + b);
   }
}
```

:white_check_mark: **GOOD** This example uses `Task.FromResult` to return the trivially computed value. It does not use any extra threads as a result.

:white_check_mark: **GOOD** 이 예제에서는 `Task.FromResult`를 사용하여 간단하게 계산된 값을 반환합니다. 결과적으로 추가 스레드를 사용하지 않습니다. 138


```C#
public class MyLibrary
{
   public Task<int> AddAsync(int a, int b)
   {
       return Task.FromResult(a + b);
   }
}
```

:bulb:**NOTE: Using `Task.FromResult` will result in a `Task` allocation. Using `ValueTask<T>` can completely remove that allocation.**

:bulb:**참고: `Task.FromResult`를 사용하면 `Task` 할당이 발생합니다. `ValueTask<T>`를 사용하면 해당 할당을 완전히 제거할 수 있습니다.** 153


:white_check_mark: **GOOD** This example uses a `ValueTask<int>` to return the trivially computed value. It does not use any extra threads as a result. It also does not allocate an object on the managed heap.

:white_check_mark: **GOOD** 이 예제에서는 `ValueTask<int>`를 사용하여 간단하게 계산된 값을 반환합니다. 결과적으로 추가 스레드를 사용하지 않습니다. 또한 관리되는 힙에 개체를 할당하지 않습니다. 158


```C#
public class MyLibrary
{
   public ValueTask<int> AddAsync(int a, int b)
   {
       return new ValueTask<int>(a + b);
   }
}
```

## Avoid using Task.Run for long running work that blocks the thread

## 스레드를 차단하는 장기 실행 작업에는 Task.Run을 사용하지 마십시오. 173


Long running work in this context refers to a thread that's running for the lifetime of the application doing background work (like processing queue items, or sleeping and waking up to process some data). 이 컨텍스트에서 장기 실행 작업은 백그라운드 작업(예: 대기열 항목 처리 또는 일부 데이터 처리를 위해 잠자기 및 깨우기)을 수행하는 애플리케이션의 수명 동안 실행되는 스레드를 나타냅니다.

 `Task.Run` will queue a work item to the thread pool. The assumption is that that work will finish quickly (or quickly enough to allow reusing that thread within some reasonable timeframe). 'Task.Run'은 작업 항목을 스레드 풀에 대기시킵니다. 작업이 빨리 완료될 것이라고 가정합니다(또는 합리적인 시간 내에 해당 스레드를 재사용할 수 있을 만큼 충분히 빨리).
 
 Stealing a thread-pool thread for long-running work is bad since it takes that thread away from other work that could be done (timer callbacks, task continuations etc). Instead, spawn a new thread manually to do long running blocking work.  장기 실행 작업을 위해 스레드 풀 스레드를 훔치는 것은 수행할 수 있는 다른 작업(타이머 콜백, 작업 연속 등)에서 해당 스레드를 가져오기 때문에 좋지 않습니다. 대신, 장기 실행 차단 작업을 수행하려면 새 스레드를 수동으로 생성하십시오. 182


:bulb: **NOTE: The thread pool grows if you block threads but it's bad practice to do so.**

:bulb: **참고: 스레드를 차단하면 스레드 풀이 커지지만 그렇게 하는 것은 좋지 않습니다.** 185


:bulb: **NOTE:`Task.Factory.StartNew` has an option `TaskCreationOptions.LongRunning` that under the covers creates a new thread and returns a Task that represents the execution. Using this properly requires several non-obvious parameters to be passed in to get the right behavior on all platforms.**

참고:`Task.Factory.StartNew`에는 새 스레드를 만들고 실행을 나타내는 작업을 반환하는 `TaskCreationOptions.LongRunning` 옵션이 있습니다. 이것을 올바르게 사용하려면 모든 플랫폼에서 올바른 동작을 얻기 위해 몇 가지 명확하지 않은 매개변수를 전달해야 합니다.**


:bulb: **NOTE: Don't use `TaskCreationOptions.LongRunning` with async code as this will create a new thread which will be destroyed after first `await`.**

:bulb: **참고: 비동기 코드와 함께 `TaskCreationOptions.LongRunning`을 사용하지 마십시오. 이렇게 하면 첫 번째 `await` 후에 소멸될 새 스레드가 생성됩니다.** 195



❌ **BAD** This example steals a thread-pool thread forever, to execute queued work on a `BlockingCollection<T>`.

❌ **BAD** 이 예제는 'BlockingCollection<T>'에서 대기 중인 작업을 실행하기 위해 스레드 풀 스레드를 영원히 훔칩니다. 201


```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        Task.Run(ProcessQueue);
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

:white_check_mark: **GOOD** This example uses a dedicated thread to process the message queue instead of a thread-pool thread.
 
 :white_check_mark: **GOOD** 이 예에서는 스레드 풀 스레드 대신 전용 스레드를 사용하여 메시지 큐를 처리합니다. 233


```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        var thread = new Thread(ProcessQueue) 
        {
            // This is important as it allows the process to exit while this thread is running
            IsBackground = true
        };
        thread.Start();
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

## Avoid using `Task.Result` and `Task.Wait`
 `Task.Result` 및 `Task.Wait` 사용을 피하세요.


There are very few ways to use `Task.Result` and `Task.Wait` correctly so the general advice is to completely avoid using them in your code. 

 `Task.Result` 및 `Task.Wait`를 올바르게 사용하는 방법은 거의 없으므로 코드에서 사용하지 않는 것이 일반적입니다. 274


### :warning: Sync over `async`


Using `Task.Result` or `Task.Wait` to block wait on an asynchronous operation to complete is *MUCH* worse than calling a truly synchronous API to block. This phenomenon is dubbed "Sync over async". Here is what happens at a very high level:
 
 'Task.Result' 또는 'Task.Wait'를 사용하여 비동기 작업이 완료될 때까지 대기를 차단하는 것은 진정한 동기 API를 호출하여 차단하는 것보다 *훨씬* 나쁩니다. 이 현상을 "동기화를 통한 동기화"라고 합니다. 다음은 매우 높은 수준에서 발생하는 일입니다. 282


- An asynchronous operation is kicked off. - 비동기 작업이 시작됩니다. 287
- The calling thread is blocked waiting for that operation to complete. - 호출 스레드는 해당 작업이 완료되기를 기다리는 동안 차단됩니다. 
- When the asynchronous operation completes, it unblocks the code waiting on that operation. This takes place on another thread. 비동기 작업이 완료되면 해당 작업을 기다리고 있는 코드의 차단을 해제합니다. 이것은 다른 스레드에서 발생합니다.

The result is that we need to use 2 threads instead of 1 to complete synchronous operations. This usually leads to [thread-pool starvation](https://blogs.msdn.microsoft.com/vancem/2018/10/16/diagnosing-net-core-threadpool-starvation-with-perfview-why-my-service-is-not-saturating-all-cores-or-seems-to-stall/) and results in service outages.
 
결과는 동기 작업을 완료하기 위해 1개 대신 2개의 스레드를 사용해야 한다는 것입니다. 이것은 일반적으로 [스레드 풀 기아]로 이어집니다.


### :warning: Deadlocks
 경고 데드락

The `SynchronizationContext` is an abstraction that gives application models a chance to control where asynchronous continuations run. ASP.NET (non-core), WPF and Windows Forms each have an implementation that will result in a deadlock if Task.Wait or Task.Result is used on the main thread. This behavior has led to a bunch of "clever" code snippets that show the "right" way to block waiting for a Task. The truth is, there's no good way to block waiting for a Task to complete.
 
 'SynchronizationContext'는 애플리케이션 모델에 비동기 연속 실행이 실행되는 위치를 제어할 수 있는 기회를 제공하는 추상화입니다. ASP.NET(비코어), WPF 및 Windows Forms에는 각각 Task.Wait 또는 Task.Result가 기본 스레드에서 사용되는 경우 교착 상태가 발생하는 구현이 있습니다. 이 동작으로 인해 작업 대기를 차단하는 "올바른" 방법을 보여주는 "영리한" 코드 조각이 많이 만들어졌습니다. 사실 태스크가 완료될 때까지 기다리는 것을 차단하는 좋은 방법은 없습니다.


:bulb:**NOTE: ASP.NET Core does not have a `SynchronizationContext` and is not prone to the deadlock problem.**
 :bulb:**참고: ASP.NET Core에는 'SynchronizationContext'가 없으며 교착 상태 문제가 발생하지 않습니다.** 304


❌ **BAD** The below are all examples that are, in one way or another, trying to avoid the deadlock situation but still succumb to "sync over async" problems.
 ❌ **나쁜** 다음은 어떤 식으로든 교착 상태를 피하려고 하지만 여전히 "비동기를 통한 동기화" 문제에 굴복하는 모든 예입니다. 308


```C#
public string DoOperationBlocking()
{
    // Bad - Blocking the thread that enters.
    // DoAsyncOperation will be scheduled on the default task scheduler, and remove the risk of deadlocking.
    // In the case of an exception, this method will throw an AggregateException wrapping the original exception.
    return Task.Run(() => DoAsyncOperation()).Result;
}

public string DoOperationBlocking2()
{
    // Bad - Blocking the thread that enters.
    // DoAsyncOperation will be scheduled on the default task scheduler, and remove the risk of deadlocking.
    // In the case of an exception, this method will throw the exception without wrapping it in an AggregateException.
    return Task.Run(() => DoAsyncOperation()).GetAwaiter().GetResult();
}

public string DoOperationBlocking3()
{
    // Bad - Blocking the thread that enters, and blocking the threadpool thread inside.
    // In the case of an exception, this method will throw an AggregateException containing another AggregateException, containing the original exception.
    return Task.Run(() => DoAsyncOperation().Result).Result;
}

public string DoOperationBlocking4()
{
    // Bad - Blocking the thread that enters, and blocking the threadpool thread inside.
    return Task.Run(() => DoAsyncOperation().GetAwaiter().GetResult()).GetAwaiter().GetResult();
}

public string DoOperationBlocking5()
{
    // Bad - Blocking the thread that enters.
    // Bad - No effort has been made to prevent a present SynchonizationContext from becoming deadlocked.
    // In the case of an exception, this method will throw an AggregateException wrapping the original exception.
    return DoAsyncOperation().Result;
}

public string DoOperationBlocking6()
{
    // Bad - Blocking the thread that enters.
    // Bad - No effort has been made to prevent a present SynchonizationContext from becoming deadlocked.
    return DoAsyncOperation().GetAwaiter().GetResult();
}

public string DoOperationBlocking7()
{
    // Bad - Blocking the thread that enters.
    // Bad - No effort has been made to prevent a present SynchonizationContext from becoming deadlocked.
    var task = DoAsyncOperation();
    task.Wait();
    return task.GetAwaiter().GetResult();
}
```

## Prefer `await` over `ContinueWith`

`Task` existed before the async/await keywords were introduced and as such provided ways to execute continuations without relying on the language. Although these methods are still valid to use, we generally recommend that you prefer `async`/`await` to using `ContinueWith`. `ContinueWith` also does not capture the `SynchronizationContext` and as a result is actually semantically different to `async`/`await`.
 
 'Task'는 async/await 키워드가 도입되기 전에 존재했으며 언어에 의존하지 않고 연속 작업을 실행할 수 있는 방법을 제공했습니다. 이러한 방법은 여전히 유효하지만 일반적으로 `ContinueWith`를 사용하는 것보다 `async`/`await`를 선호하는 것이 좋습니다. `ContinueWith`는 `SynchronizationContext`도 캡처하지 않으며 결과적으로 실제로 `async`/`await`와 의미상 다릅니다.


❌ **BAD** The example uses `ContinueWith` instead of `async`
❌ **BAD** 이 예에서는 `async` 대신 `ContinueWith`를 사용합니다. 374

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync().ContinueWith(task =>
    {
        return task.Result + 1;
    });
}
```

:white_check_mark: **GOOD** This example uses the `await` keyword to get the result from `CallDependencyAsync`.
 :white_check_mark: **GOOD** 이 예는 `await` 키워드를 사용하여 `CallDependencyAsync`에서 결과를 가져옵니다. 387


```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Always create `TaskCompletionSource<T>` with `TaskCreationOptions.RunContinuationsAsynchronously`

`TaskCompletionSource<T>` is an important building block for libraries trying to adapt things that are not inherently awaitable to be awaitable via a `Task`. It is also commonly used to build higher-level operations (such as batching and other combinators) on top of existing asynchronous APIs. By default, `Task` continuations will run *inline* on the same thread that calls Try/Set(Result/Exception/Canceled). As a library author, this means having to understand that calling code can resume directly on your thread. This is extremely dangerous and can result in deadlocks, thread-pool starvation, corruption of state (if code runs unexpectedly) and more. 
 
 `TaskCompletionSource<T>`는 본질적으로 대기할 수 없는 것을 `Task`를 통해 대기할 수 있도록 조정하려는 라이브러리의 중요한 빌딩 블록입니다. 또한 기존 비동기 API 위에 더 높은 수준의 작업(예: 일괄 처리 및 기타 결합기)을 빌드하는 데 일반적으로 사용됩니다. 기본적으로 `Task` 연속은 Try/Set(Result/Exception/Canceled)를 호출하는 동일한 스레드에서 *인라인*으로 실행됩니다. 라이브러리 작성자로서 이는 호출 코드가 스레드에서 직접 재개될 수 있음을 이해해야 함을 의미합니다. 이것은 매우 위험하며 교착 상태, 스레드 풀 기아, 상태 손상(코드가 예기치 않게 실행되는 경우) 등을 초래할 수 있습니다.

Always use `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`. This will dispatch the continuation onto the thread pool instead of executing it inline.
 
`TaskCompletionSource<T>`를 생성할 때는 항상 `TaskCreationOptions.RunContinuationsAsynchronously`를 사용하세요. 이렇게 하면 인라인으로 실행하는 대신 스레드 풀에 연속 작업을 디스패치합니다. 

❌ **BAD** This example does not use `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`.
 
 ❌ **BAD** 이 예제에서는 `TaskCompletionSource<T>`를 생성할 때 `TaskCreationOptions.RunContinuationsAsynchronously`를 사용하지 않습니다. 409

```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>();
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on this thread!
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

:white_check_mark: **GOOD** This example uses `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`.
 
 :white_check_mark: **GOOD** 이 예제에서는 `TaskCompletionSource<T>`를 생성할 때 `TaskCreationOptions.RunContinuationsAsynchronously`를 사용합니다. 429


```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on a different thread-pool thread
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

:bulb:**NOTE:비슷하게 보이는 2개의 열거형이 있습니다. There are 2 enums that look alike. [`TaskCreationOptions.RunContinuationsAsynchronously`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcreationoptions?view=netcore-2.0#System_Threading_Tasks_TaskCreationOptions_RunContinuationsAsynchronously) and [`TaskContinuationOptions.RunContinuationsAsynchronously`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcontinuationoptions?view=netcore-2.0). Be careful not to confuse their usage.** 
 사용법을 혼동하지 않도록 주의하세요.

## Always dispose `CancellationTokenSource`(s) used for timeouts
## 시간 초과에 사용된 `CancellationTokenSource`를 항상 폐기합니다. 453


`CancellationTokenSource` objects that are used for timeouts (are created with timers or uses the `CancelAfter` method), can put pressure on the timer queue if not disposed.
타임아웃에 사용되는 'CancellationTokenSource' 개체(타이머로 생성되거나 'CancelAfter' 메서드 사용)는 삭제되지 않으면 타이머 대기열에 압력을 가할 수 있습니다.


❌ **BAD** This example does not dispose the `CancellationTokenSource` and as a result the timer stays in the queue for 10 seconds after each request is made.
❌ **BAD** 이 예는 'CancellationTokenSource'를 삭제하지 않으며 결과적으로 타이머는 각 요청이 이루어진 후 10초 동안 대기열에 머뭅니다. 461


```C#
public async Task<Stream> HttpClientAsyncWithCancellationBad()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    using (var client = _httpClientFactory.CreateClient())
    {
        var response = await client.GetAsync("http://backend/api/1", cts.Token);
        return await response.Content.ReadAsStreamAsync();
    }
}
```

:white_check_mark: **GOOD** This example disposes the `CancellationTokenSource` and properly removes the timer from the queue.
:white_check_mark: **GOOD** 이 예는 'CancellationTokenSource'를 삭제하고 대기열에서 타이머를 적절하게 제거합니다. 478


```C#
public async Task<Stream> HttpClientAsyncWithCancellationGood()
{
    using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)))
    {
        using (var client = _httpClientFactory.CreateClient())
        {
            var response = await client.GetAsync("http://backend/api/1", cts.Token);
            return await response.Content.ReadAsStreamAsync();
        }
    }
}
```

## Always flow `CancellationToken`(s) to APIs that take a `CancellationToken`
## 항상 'CancellationToken'을 사용하는 API에 'CancellationToken'을 전달합니다. 496


Cancellation is cooperative in .NET. Everything in the call-chain has to be explicitly passed the `CancellationToken` in order for it to work well. This means you need to explicitly pass the token into other APIs that take a token if you want cancellation to be most effective.
 
 취소는 .NET에서 협조적입니다. 콜 체인의 모든 것이 제대로 작동하려면 'CancellationToken'을 명시적으로 전달해야 합니다. 즉, 취소가 가장 효과적이려면 토큰을 사용하는 다른 API에 토큰을 명시적으로 전달해야 합니다.

❌ **BAD** This example neglects to pass the `CancellationToken` to `Stream.ReadAsync` making the operation effectively not cancellable.
 
 ❌ **BAD** 이 예시는 `CancellationToken`을 `Stream.ReadAsync`에 전달하는 것을 무시하여 작업을 사실상 취소할 수 없도록 만듭니다. 504

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   // We forgot to pass flow cancellationToken to ReadAsync
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

:white_check_mark: **GOOD** This example passes the `CancellationToken` into `Stream.ReadAsync`.

:white_check_mark: **GOOD** 이 예시는 `CancellationToken`을 `Stream.ReadAsync`에 전달합니다. 518

 
```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   // This properly flows cancellationToken to ReadAsync
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

## Cancelling uncancellable operations
 ## 취소할 수 없는 작업 취소 533


One of the coding patterns that appears when doing asynchronous programming is cancelling an uncancellable operation. This usually means creating another task that completes when a timeout or `CancellationToken` fires, and then using `Task.WhenAny` to detect a complete or cancelled operation.
 
 비동기 프로그래밍을 수행할 때 나타나는 코딩 패턴 중 하나는 취소할 수 없는 작업을 취소하는 것입니다. 이는 일반적으로 타임아웃 또는 `CancellationToken`이 실행될 때 완료되는 다른 작업을 생성한 다음 `Task.WhenAny`를 사용하여 완료되거나 취소된 작업을 감지하는 것을 의미합니다.

### Using CancellationTokens

❌ **BAD** This example uses `Task.Delay(-1, token)` to create a `Task` that completes when the `CancellationToken` fires, but if it doesn't fire, there's no way to dispose the `CancellationTokenRegistration`. This can lead to a memory leak.
 
 ❌ **BAD** 이 예제는 `Task.Delay(-1, token)`를 사용하여 `CancellationToken`이 실행될 때 완료되는 `Task`를 생성하지만 실행되지 않으면 ` CancellationTokenRegistration`. 이로 인해 메모리 누수가 발생할 수 있습니다. 543


```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    // There's no way to dispose the registration
    var delayTask = Task.Delay(-1, cancellationToken);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

:white_check_mark: **GOOD** This example disposes the `CancellationTokenRegistration` when one of the `Task(s)` complete.
 
 :white_check_mark: **GOOD** 이 예는 `Task(s)` 중 하나가 완료되면 `CancellationTokenRegistration`을 삭제합니다. 565


```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    var tcs = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);

    // This disposes the registration as soon as one of the tasks trigger
    using (cancellationToken.Register(state =>
    {
        ((TaskCompletionSource<object>)state).TrySetResult(null);
    },
    tcs))
    {
        var resultTask = await Task.WhenAny(task, tcs.Task);
        if (resultTask == tcs.Task)
        {
            // Operation cancelled
            throw new OperationCanceledException(cancellationToken);
        }

        return await task;
    }
}
```

### Using a timeout

❌ **BAD** This example does not cancel the timer even if the operation successfully completes. This means you could end up with lots of timers, which can flood the timer queue. 
 
 ❌ **BAD** 이 예제는 작업이 성공적으로 완료되어도 타이머를 취소하지 않습니다. 즉, 타이머 대기열이 플러딩될 수 있는 많은 타이머가 발생할 수 있습니다. 596


```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    var delayTask = Task.Delay(timeout);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

:white_check_mark: **GOOD** This example cancels the timer if the operation successfully completes.
 
 :white_check_mark: **GOOD** 이 예는 작업이 성공적으로 완료되면 타이머를 취소합니다. 617


```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    using (var cts = new CancellationTokenSource())
    {
        var delayTask = Task.Delay(timeout, cts.Token);

        var resultTask = await Task.WhenAny(task, delayTask);
        if (resultTask == delayTask)
        {
            // Operation cancelled
            throw new OperationCanceledException();
        }
        else
        {
            // Cancel the timer task so that it does not fire
            cts.Cancel();
        }

        return await task;
    }
}
```

## Always call `FlushAsync` on `StreamWriter`(s) or `Stream`(s) before calling `Dispose`
 ## 'Dispose'를 호출하기 전에 항상 'StreamWriter' 또는 'Stream'에서 'FlushAsync'를 호출하세요. 646


When writing to a `Stream` or `StreamWriter`, even if the asynchronous overloads are used for writing, the underlying data might be buffered. When data is buffered, disposing the `Stream` or `StreamWriter` via the `Dispose` method will synchronously write/flush, which results in blocking the thread and could lead to thread-pool starvation. Either use the asynchronous `DisposeAsync` method (for example via `await using`) or call `FlushAsync` before calling `Dispose`.
 
 'Stream' 또는 'StreamWriter'에 쓸 때 쓰기에 비동기 오버로드가 사용되더라도 기본 데이터가 버퍼링될 수 있습니다. 데이터가 버퍼링될 때 'Dispose' 메서드를 통해 'Stream' 또는 'StreamWriter'를 삭제하면 동기적으로 쓰기/플러시되어 스레드가 차단되고 스레드 풀 기아가 발생할 수 있습니다. 비동기식 `DisposeAsync` 메서드를 사용하거나(예: `await using`을 통해) `Dispose`를 호출하기 전에 `FlushAsync`를 호출하세요.

:bulb:**NOTE: This is only problematic if the underlying subsystem does IO.**
 :bulb:**참고: 기본 하위 시스템이 IO를 수행하는 경우에만 문제가 됩니다.** 654


❌ **BAD** This example ends up blocking the request by writing synchronously to the HTTP-response body.
 ❌ **BAD** 이 예제는 HTTP 응답 본문에 동기적으로 작성하여 요청을 차단합니다. 658


```C#
app.Run(async context =>
{
    // The implicit Dispose call will synchronously write to the response body
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** This example asynchronously flushes any buffered data while disposing the `StreamWriter`.
 :white_check_mark: **GOOD** 이 예는 `StreamWriter`를 삭제하는 동안 버퍼링된 모든 데이터를 비동기식으로 플러시합니다. 673


```C#
app.Run(async context =>
{
    // The implicit AsyncDispose call will flush asynchronously
    await using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** This example asynchronously flushes any buffered data before disposing the `StreamWriter`.
 :white_check_mark: **GOOD** 이 예는 `StreamWriter`를 삭제하기 전에 버퍼링된 모든 데이터를 비동기식으로 플러시합니다. 688


```C#
app.Run(async context =>
{
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");

        // Force an asynchronous flush
        await streamWriter.FlushAsync();
    }
});
```

## Prefer `async`/`await` over directly returning `Task`
 ## `Task`를 직접 반환하는 것보다 `async` /`를 선호합니다. 705


There are benefits to using the `async`/`await` keyword instead of directly returning the `Task`:
- Asynchronous and synchronous exceptions are normalized to always be asynchronous.
- The code is easier to modify (consider adding a `using`, for example).
- Diagnostics of asynchronous methods are easier (debugging hangs etc).
- Exceptions thrown will be automatically wrapped in the returned `Task` instead of surprising the caller with an actual exception.
- Async locals will not leak out of async methods. If you set an async local in a non-async method, it will "leak" out of that call.
 
`Task`를 직접 반환하는 대신 `async`/`await` 키워드를 사용하면 이점이 있습니다.
- 비동기 및 동기 예외는 항상 비동기로 정규화됩니다.
- 코드 수정이 더 쉽습니다(예: 'using' 추가 고려).
- 비동기 방식의 진단이 더 쉽습니다(디버깅 정지 등).
- throw된 예외는 실제 예외로 호출자를 놀라게 하는 대신 반환된 'Task'에 자동으로 래핑됩니다.
- 비동기 로컬은 비동기 메서드에서 누출되지 않습니다. 비동기가 아닌 메서드에서 비동기 로컬을 설정하면 해당 호출에서 "누출"됩니다. 

❌ **BAD** This example directly returns the `Task` to the caller.
❌ **BAD** 이 예제는 'Task'를 호출자에게 직접 반환합니다. 723

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync();
}
```

:white_check_mark: **GOOD** This examples uses async/await instead of directly returning the Task.
:white_check_mark: **GOOD** 이 예제에서는 Task를 직접 반환하는 대신 async/await를 사용합니다. 733


```C#
public async Task<int> DoSomethingAsync()
{
    return await CallDependencyAsync();
}
```

:bulb:**NOTE: There are performance considerations when using an async state machine over directly returning the `Task`. It's always faster to directly return the `Task` since it does less work but you end up changing the behavior and potentially losing some of the benefits of the async state machine.**
 
 :bulb:**참고: 'Task'를 직접 반환하는 것보다 비동기 상태 머신을 사용할 때 성능 고려 사항이 있습니다. 'Task'를 직접 반환하는 것은 작업량이 적기 때문에 항상 더 빠르지만 결국 동작을 변경하고 잠재적으로 비동기 상태 머신의 일부 이점을 잃게 됩니다.**

## ConfigureAwait

TBD

# Scenarios

The above tries to distill general guidance, but doesn't do justice to the kinds of real-world situations that cause code like this to be written in the first place (bad code). This section tries to take concrete examples from real applications and turn them into something simple to help you relate these problems to existing codebases.
 
 위의 내용은 일반적인 지침을 추출하려고 시도하지만 처음부터 이와 같은 코드(나쁜 코드)를 작성하게 하는 실제 상황의 종류를 정의하지 않습니다. 이 섹션에서는 실제 응용 프로그램에서 구체적인 예를 가져와 이러한 문제를 기존 코드베이스와 연관시키는 데 도움이 되도록 간단한 것으로 바꾸려고 합니다.

## `Timer` callbacks

❌ **BAD** The `Timer` callback is `void`-returning and we have asynchronous work to execute. This example uses `async void` to accomplish it and as a result can crash the process if an exception occurs.
 
 ❌ **BAD** `Timer` 콜백은 `void`를 반환하며 실행할 비동기 작업이 있습니다. 이 예제는 'async void'를 사용하여 이를 수행하며 결과적으로 예외가 발생하면 프로세스가 중단될 수 있습니다.


```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public async void Heartbeat(object state)
    {
        await _client.GetAsync("http://mybackend/api/ping");
    }
}
```

❌ **BAD** This attempts to block in the `Timer` callback. This may result in thread-pool starvation and is an example of [sync over async](#warning-sync-over-async)
 
 ❌ **BAD** 이것은 `Timer` 콜백에서 차단을 시도합니다. 이것은 스레드 풀 기아를 초래할 수 있으며 [sync over async](#warning-sync-over-async)의 예입니다. 784


```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        _client.GetAsync("http://mybackend/api/ping").GetAwaiter().GetResult();
    }
}
```

:white_check_mark: **GOOD** This example uses an `async Task`-based method and discards the `Task` in the `Timer` callback. If this method fails, it will not crash the process. :white_check_mark: **GOOD** 이 예제는 `async Task` 기반 메소드를 사용하고 `Timer` 콜백에서 `Task`를 버립니다. 이 방법이 실패하면 프로세스가 중단되지 않습니다.
 
 Instead, it will fire the [`TaskScheduler.UnobservedTaskException`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception?view=netframework-4.7.2) event. 대신 이걸 참조하세요.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        // Discard the result
        _ = DoAsyncPing();
    }

    private async Task DoAsyncPing()
    {
        await _client.GetAsync("http://mybackend/api/ping");
    }
}
```

## Implicit `async void` delegates
 ## 암시적 `async void` 대리자 837


Imagine a `BackgroundQueue` with a `FireAndForget` that takes a callback. This method will execute the callback at some time in the future.
 
 콜백을 받는 `FireAndForget`이 있는 `BackgroundQueue`를 상상해 보세요. 이 메소드는 미래의 어느 시점에서 콜백을 실행할 것입니다. 841


❌ **BAD** This will force callers to either block in the callback or use an `async void` delegate.
 
 ❌ **나쁜** 이것은 호출자가 콜백에서 차단하거나 `async void` 대리자를 사용하도록 강제합니다. 846


```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
}
```

❌ **BAD** This calling code is creating an `async void` method implicitly. The compiler fully supports this today.
 
 ❌ **BAD** 이 호출 코드는 암시적으로 `async void` 메서드를 생성합니다. 컴파일러는 오늘날 이를 완벽하게 지원합니다. 858


```C#
public class Program
{
    public void Main(string[] args)
    {
        var httpClient = new HttpClient();
        BackgroundQueue.FireAndForget(async () =>
        {
            await httpClient.GetAsync("http://pinger/api/1");
        });
        
        Console.ReadLine();
    }
}
```

:white_check_mark: **GOOD** This BackgroundQueue implementation offers both sync and `async` callback overloads.
 
 :white_check_mark: **GOOD** 이 BackgroundQueue 구현은 동기화 및 '비동기' 콜백 오버로드를 모두 제공합니다. 879


```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
    public static void FireAndForget(Func<Task> action) { }
}
```

## `ConcurrentDictionary.GetOrAdd`

It's pretty common to cache the result of an asynchronous operation and `ConcurrentDictionary` is a good data structure for doing that. `GetOrAdd` is a convenience API for trying to get an item if it's already there or adding it if it isn't. The callback is synchronous so it's tempting to write code that uses `Task.Result` to produce the value of an asynchronous process but that can lead to thread-pool starvation.
 
 비동기 작업의 결과를 캐시하는 것은 매우 일반적이며 'ConcurrentDictionary'는 이를 수행하는 데 좋은 데이터 구조입니다. 'GetOrAdd'는 항목이 이미 있는 경우 항목을 가져오거나 없는 경우 추가하는 편리한 API입니다. 콜백은 동기식이므로 'Task.Result'를 사용하여 비동기식 프로세스의 값을 생성하는 코드를 작성하고 싶지만 스레드 풀 기아로 이어질 수 있습니다.

❌ **BAD** This may result in thread-pool starvation since we're blocking the request thread if the person data is not cached.
 
 ❌ **나쁜** 개인 데이터가 캐시되지 않은 경우 요청 스레드를 차단하므로 스레드 풀 기아가 발생할 수 있습니다. 898


```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Person> _cache = new ConcurrentDictionary<int, Person>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public IActionResult Get(int id)
   {
       var person = _cache.GetOrAdd(id, (key) => _db.People.FindAsync(key).Result);
       return Ok(person);
   }
}
```

:white_check_mark: **GOOD** This implementation won't result in thread-pool starvation since we're storing a task instead of the result itself.
 
 :white_check_mark: **GOOD** 이 구현은 결과 자체 대신 작업을 저장하기 때문에 스레드 풀 기아를 초래하지 않습니다. 924


:warning: `ConcurrentDictionary.GetOrAdd`, when accessed concurrently, may run the value-constructing delegate multiple times. This can result in needlessly kicking off the same potentially expensive computation multiple times.
 
 :경고: `ConcurrentDictionary.GetOrAdd`는 동시에 액세스할 때 값 구성 대리자를 여러 번 실행할 수 있습니다. 이로 인해 잠재적으로 비용이 많이 드는 동일한 계산을 여러 번 불필요하게 시작해야 할 수 있습니다. 929


```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Task<Person>> _cache = new ConcurrentDictionary<int, Task<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => _db.People.FindAsync(key));
       return Ok(person);
   }
}
```

:white_check_mark: **GOOD** This implementation prevents the delegate from being executed multiple times, by using the `async` lazy pattern: even if construction of the AsyncLazy instance happens multiple times ("cheap" operation), the delegate will be called only once.
 
 :white_check_mark: **GOOD** 이 구현은 `async` 지연 패턴을 사용하여 대리자가 여러 번 실행되는 것을 방지합니다. AsyncLazy 인스턴스 생성이 여러 번 발생하더라도("저렴한" 작업) 대리자가 호출됩니다. 한 번만.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, AsyncLazy<Person>> _cache = new ConcurrentDictionary<int, AsyncLazy<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => new AsyncLazy<Person>(() => _db.People.FindAsync(key))).Value;
       return Ok(person);
   }
   
   private class AsyncLazy<T> : Lazy<Task<T>>
   {
      public AsyncLazy(Func<Task<T>> valueFactory) : base(valueFactory)
      {
      }
   }
}
```

## Constructors

Constructors are synchronous. If you need to initialize some logic that may be asynchronous, there are a couple of patterns for dealing with this.
 
 생성자는 동기식입니다. 비동기일 수 있는 일부 논리를 초기화해야 하는 경우 이를 처리하기 위한 몇 가지 패턴이 있습니다. 989


Here's an example of using a client API that needs to connect asynchronously before use.
 
 다음은 사용하기 전에 비동기식으로 연결해야 하는 클라이언트 API를 사용하는 예입니다. 994


```C#
public interface IRemoteConnectionFactory
{
   Task<IRemoteConnection> ConnectAsync();
}

public interface IRemoteConnection
{
    Task PublishAsync(string channel, string message);
    Task DisposeAsync();
}
```


❌ **BAD** This example uses `Task.Result` to get the connection in the constructor. This could lead to thread-pool starvation and deadlocks.
 
 ❌ **BAD** 이 예제는 `Task.Result`를 사용하여 생성자에서 연결을 가져옵니다. 이로 인해 스레드 풀 기아 및 교착 상태가 발생할 수 있습니다. 1013


```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;
    
    public Service(IRemoteConnectionFactory connectionFactory)
    {
        _connection = connectionFactory.ConnectAsync().Result;
    }
}
```

:white_check_mark: **GOOD** This implementation uses a static factory pattern in order to allow asynchronous construction:
 
 :white_check_mark: **GOOD** 이 구현은 비동기식 구성을 허용하기 위해 정적 팩토리 패턴을 사용합니다. 1030


```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;

    private Service(IRemoteConnection connection)
    {
        _connection = connection;
    }

    public static async Task<Service> CreateAsync(IRemoteConnectionFactory connectionFactory)
    {
        return new Service(await connectionFactory.ConnectAsync());
    }
}
```

## WindowsIdentity.RunImpersonated

This API runs the specified action as the impersonated Windows identity. An [asynchronous version of the callback](https://docs.microsoft.com/en-us/dotnet/api/system.security.principal.windowsidentity.runimpersonatedasync) was introduced in .NET 5.0.
 
 이 API는 지정된 작업을 가장한 Windows ID로 실행합니다. [콜백의 비동기 버전](https://docs.microsoft.com/en-us/dotnet/api/system.security.principal.windowsidentity.runimpersonatedasync)이 .NET 5.0에 도입되었습니다.

❌ **BAD** This example tries to execute the query asynchronously, and then wait for it outside of the call to `RunImpersonated`. This will throw because the query might be executing outside of the impersonation context.
 
 ❌ **BAD** 이 예제는 쿼리를 비동기적으로 실행하려고 시도한 다음 `RunImpersonated` 호출 외부에서 대기합니다. 쿼리가 가장 컨텍스트 외부에서 실행 중일 수 있으므로 이 오류가 발생합니다. 1058


```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    Task<IEnumerable<Product>> products = null;
    WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        context =>
        {
            products = _db.QueryAsync("SELECT Name from Products");
        }};
    return await products;
}
```

❌ **BAD** This example uses `Task.Result` to get the connection in the constructor. This could lead to thread-pool starvation and deadlocks.
 
 ❌ **BAD** 이 예제는 `Task.Result`를 사용하여 생성자에서 연결을 가져옵니다. 이로 인해 스레드 풀 기아 및 교착 상태가 발생할 수 있습니다. 1077


```C#
public IEnumerable<Product> GetDataImpersonated(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        context => _db.QueryAsync("SELECT Name from Products").Result);
}
```

:white_check_mark: **GOOD** This example awaits the result of `RunImpersonated` (the delegate is `Func<Task<IEnumerable<Product>>>` in this case). It is the recommended practice in frameworks earlier than .NET 5.0.
 
 :white_check_mark: **GOOD** 이 예제는 `RunImpersonated`의 결과를 기다립니다(이 경우 대리자는 `Func<Task<IEnumerable<Product>>>`입니다). .NET 5.0 이전의 프레임워크에서 권장되는 방법입니다. 1091


```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return await WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle, 
        context => _db.QueryAsync("SELECT Name from Products"));
}
```

:white_check_mark: **GOOD** This example uses the asynchronous `RunImpersonatedAsync` function and awaits its result. It is available in .NET 5.0 or newer.
 
 :white_check_mark: **GOOD** 이 예는 비동기 'RunImpersonatedAsync' 함수를 사용하고 결과를 기다립니다. .NET 5.0 이상에서 사용할 수 있습니다. 1105


```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return await WindowsIdentity.RunImpersonatedAsync(
        safeAccessTokenHandle, 
        context => _db.QueryAsync("SELECT Name from Products"));
}
```
