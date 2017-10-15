# 重试模式

通过透明地重试失败的操作，让应用程序可以在尝试连接到服务或网络资源时处理出现的瞬态故障。这可以提高应用程序的稳定性。

## 背景和问题

与云中运行的元素进行通信的应用程序必须对环境中可能发生的瞬态故障敏感。故障包括与组件和服务的网络连接的瞬时丢失，服务的暂时不可用或服务繁忙时发生的超时。
这些故障通常可以自我纠正，如果触发故障的动作在适当的延迟之后重复，则可能会成功。例如，处理大量并发请求的数据库服务可以实现一种限制策略，临时拒绝任何进一步的请求，直到其工作负载松动。尝试访问数据库的应用程序可能无法连接，但如果在延迟后再次尝试可能会成功。

## 解决方案

在云中，瞬态故障并不罕见，应用程序应应通过优雅而透明地设计处理。最大限度地减少故障对应用程序执行的业务任务的影响。

如果应用程序在尝试向远程服务发送请求时检测到故障，则可以使用以下策略来处理该故障：
* **取消**。如果故障不是暂时的，或者即便重复也不太可能成功，应用程序应该取消操作并报告异常。例如，由于无效凭据而导致的身份验证失败无论尝试多少次，都不可能成功。
* **重试**。如果报告的特定故障不常见，可能是由于网络数据包在发送时被破坏的异常情况引起的。在这种情况下，应用程序可以立即重试失败的请求，因为同样的错误不太可能重复，并且请求可能会成功。
* **延迟后重试**。如果故障是由更常见的连接或繁忙导致，则网络或服务可能需要较短的时间，来更正连接问题或清除累积的工作负载。应用程序应等待适当的时间才能重试请求。
对于更常见的瞬态故障，应该选择重试操作之间的时间段，以便从应用程序的多个实例中均匀地传播请求。这降低了繁忙的服务继续超载的可能。如果应用程序的许多实例不断地覆盖具有重试请求的服务，则会延长服务恢复时间。

如果请求依然失败，应用程序可以等待并再次尝试。如有必要，可以在重试尝试之间随着延迟的增加重复该过程，直到达到请求数量的最大值。延迟可以逐渐增加或指数地增加，这取决于故障类型和在此期间将更正的概率。
下图说明了使用此模式调用托管服务中的操作。如果请求在预定义的尝试次数后失败，应用程序应将故障视为异常并做相应处理。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/retry-pattern.png)

应用程序应该包含所有尝试访问远程服务的代码，实现与之前列出的策略匹配的重试策略。发送到不同服务的请求可能会受到不同的策略的约束。一些供应商提供实现重试策略的类库，应用程序可以指定最大重试次数，重试尝试之间的时间以及其它参数。
应用程序应记录故障和故障操作的详细信息。此信息对操作者很有用。如果服务频繁不可用或繁忙，往往是因为服务已耗尽资源。可以通过扩展服务来减少这些故障的频率。例如，如果数据库服务不断重载，对数据库分区并在多个服务器分散负载可能是有帮助。
>[Microsoft Entity Framework](https://docs.microsoft.com/ef/)框架提供了重试数据库操作的功能。此外，大多数Azure服务和客户端SDK都包含重试机制。详细信息请参阅[特定服务的重试指南](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific)。

## 问题和注意事项

在决定如何实现此模式时应该考虑以下几点。
根据应用程序的业务需求和故障的性质重新调整重试策略。对于某些非关键操作，最好是快速失败，而不是重试几次，影响应用程序的吞吐量。例如，在访问远程服务的交互式Web应用程序中，最好在延迟时间较短的少量重试后就宣告失败，并向用户显示合适的提示消息（例如，“请稍后再试” ）。对于批处理应用程序，可能更适合增加重试次数，并在尝试延迟时间以指数增长。
激进的重试策略，包含尝试之间的最小延迟和大量重试时间间隔，可能会导致正在靠近或处于容量极限的繁忙服务进一步降级。如果应用程序不断尝试执行失败操作，重试策略也可能会影响应用程序的响应。
如果一个请求在大量重试后仍然失败，那么应用程序最好防止更多请求使用相同的资源，并且立即报告失败。到期时，应用程序可以暂时允许一个或多个请求，以查看它们是否成功。有关此策略的更多详细信息，请参阅[断路器模式](circuit-breaker.html)。
考虑操作是否是幂等的。如果是这样，重试本来就是安全的。否则，重试可能导致操作不止一次执行，引发意外的副作用。例如，服务可能会收到请求并成功处理，但无法发送响应。这时，假设没有收到第一个请求，重试逻辑可能会重新发送请求。
根据故障的性质，对服务的请求可能因各种原因导致不同的异常而失败。一些异常可以快速解决，而其它异常的故障持续时间较长。重试策略可以根据异常的类型调整重试之间的时间。
考虑如何重试操作，作为事务的一部分将影响事务整体一致性。微调事务操作的重试策略，最大限度地发挥成功的机会，并减少撤销所有事务步骤的需要。
确保所有重试代码都针对各种故障条件进行了全面测试。检查它不会严重影响应用程序的性能或可靠性，导致服务和资源过载，或产生竞争条件或瓶颈。
仅在理解失败操作的完整上下文的情况下才实现重试逻辑。例如，如果包含重试策略的任务调用还包含重试策略的另一个任务，则这个额外的重试层增加了处理的延迟时间。最好将低级别任务配置为快速失败，并将失败的原因报告给调用它的任务。更高级别的任务可以根据自己的策略来处理故障。
记录导致重试的所有连接失败非常重要，方便识别应用程序，服务或资源的基础问题。
调查服务或资源最有可能发生的故障，以发现其是否持久或终结。如果是这样，最好把错误当作异常。应用程序可以报告或记录异常，然后尝试通过调用替代服务（如果可用）或通过提供降级的功能来继续。有关如何检测和处理长时间故障的更多信息，请参阅[断路器模式](circuit-breaker.html)。
## 何时使用该模式

在应用程序与远程服务交互或访问远程资源时可能会遇到瞬态故障时，使用此模式。 这些故障持续时间不长，并且重复先前失败的请求可能会在以后的尝试中成功。

此模式可能不太适用以下场景：

* 故障比较持久，可能会影响应用程序的响应。重复尝试可能失败的请求，对于应用程序来说是浪费时间和资源。
* 用于处理非瞬态故障导致的失败，例如由应用程序的业务逻辑中的错误引起的内部异常。
* 作为解决系统中可扩展性问题的替代方法。如果应用程序遇到频繁的忙碌故障，这表明应当垂直扩展正在访问的服务或资源。

## 案例

这个C＃的例子代码说明了如何实现重试模式。`OperationWithBasicRetryAsync`方法（如下所示）通过`TransientOperationAsync`方法异步调用外部服务。`TransientOperationAsync`方法的实现细节和服务相关，并从示例代码中省略。

```java
private int retryCount = 3;
private readonly TimeSpan delay = TimeSpan.FromSeconds(5);

public async Task OperationWithBasicRetryAsync()
{
  int currentRetry = 0;

  for (;;)
  {
    try
    {
      // Call external service.
      await TransientOperationAsync();

      // Return or break.
      break;
    }
    catch (Exception ex)
    {
      Trace.TraceError("Operation Exception");

      currentRetry++;

      // Check if the exception thrown was a transient exception
      // based on the logic in the error detection strategy.
      // Determine whether to retry the operation, as well as how
      // long to wait, based on the retry strategy.
      if (currentRetry > this.retryCount || !IsTransient(ex))
      {
        // If this isn't a transient error or we shouldn't retry, 
        // rethrow the exception.
        throw;
      }
    }

    // Wait to retry the operation.
    // Consider calculating an exponential delay here and
    // using a strategy best suited for the operation and fault.
    await Task.Delay(delay);
  }
}

// Async method that wraps a call to a remote service (details not shown).
private async Task TransientOperationAsync()
{
  ...
}
```

调用此方法的语句包含在一个包装在for循环中的`try / catch`块中。如果`TransientOperationAsync`方法调用成功，不抛出异常，for循环将退出。如果`TransientOperationAsync`方法失败，catch块将检查失败的原因。如果判定是一个瞬态错误，代码在重试操作之前等待一个短暂的延迟。
for循环还跟踪尝试操作的次数，如果代码失败三次，则认为异常会持久。如果异常不是瞬态的或持久的，捕获处理程序会引发异常。异常退出for循环，被调用`OperationWithBasicRetryAsync`方法的代码捕获。
`IsTransient`方法（如下所示）检查与代码运行的环境相关的一组特定异常。瞬态异常的定义，因正在访问的资源和执行操作的环境而有所不同。

```java
private bool IsTransient(Exception ex)
{
  // Determine if the exception is transient.
  // In some cases this is as simple as checking the exception type, in other
  // cases it might be necessary to inspect other properties of the exception.
  if (ex is OperationTransientException)
    return true;

  var webException = ex as WebException;
  if (webException != null)
  {
    // If the web exception contains one of the following status values
    // it might be transient.
    return new[] {WebExceptionStatus.ConnectionClosed,
                  WebExceptionStatus.Timeout,
                  WebExceptionStatus.RequestCanceled }.
            Contains(webException.Status);
  }

  // Additional exception checking logic goes here.
  return false;
}
```
## 相关模式和指南

* [断路器模式](circuit-breaker.md)。重试模式对于处理瞬态故障很有用。如果预计故障会持续更长时间，则可能更适合实施断路器模式。重试模式还可以与断路器一起使用，以提供处理故障的综合方法。
* [重试针对具体服务的指南](https://docs.microsoft.com/en-us/azure/architecture/best-practices/retry-service-specific)
* [连接弹性](https://docs.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency)