## 断路器模式(Circuit Breaker)

连接到远程服务或资源发生故障时，能花一些时间来修复问题。这可以提高应用程序的稳定性和弹性。

### 背景和问题

在分布式环境中，对远程资源和服务的调用可能会由于瞬态故障而失败，例如网络连接速度慢，超时，资源过剩或暂时不可用。这些故障通常会在短时间内自行修正，鲁棒性好的云应用程序应当使用诸如[重试模式](retry.html)的策略来准备好处理它们。

但是，也可能会出现意外事件造成的故障，而且修复可能需要更长时间。这些故障的严重程度可以从部分连接失效到服务完全失效。在这些情况下，应用可能会无意义地重试不太可能成功的操作，而实际上应用应该快速接受操作失败并相应地处理此故障。

另外，如果服务非常繁忙，系统一部分的故障可能会导致级联故障。例如，调用服务的操作可以配置为实现超时，如果服务在该时间段内没有响应，则回复失败消息。但是，该策略可能会阻止许多并发请求，直到超时期限到期。这些被阻止的请求可能会持有关键的系统资源，如内存，线程，数据库连接等。因此，这些资源可能会耗尽，导致系统中其它无关的，需要使用相同的资源的部分故障。在这种情况下，最好立即执行操作，只有在可能成功的情况下才尝试调用该服务。请注意，设置较短的超时可能有助于解决此问题，但超时时间不应太短，以至于操作在大多数时候失败，即使对服务的请求最终会成功。

## 解决方案

断路器模式可以防止应用程序重复尝试执行可能失败的操作。允许它继续，而不必等待故障修复或浪费CPU周期，同时判断故障是否持续。断路器模式让应用程序能够检测故障是否已解决。如果问题已经修复，应用程序可以尝试调用该操作。

> 断路器模式的目的不同于重试模式。重试模式使应用程序可以重试一个操作，期望它会成功。断路器模式可防止应用程序执行可能失败的操作。应用程序可以将两者结合起来，使用重试模式通过断路器执行操作。然而，重试逻辑应对断路器返回的任何异常敏感，如果断路器指示故障不是瞬态，则放弃重试尝试。

断路器作为可能失败的操作的代理。应监视最近发生的故障数，并使用此信息来决定是否允许操作继续，或者立即返回异常。

代理可以被实现为具有模拟电路断路器功能的状态机，包含以下的状态：

**关闭**：应用程序的请求被路由到操作。代理维护最近的故障数量的计数，如果对操作的调用不成功，则代理会增加此计数。如果最近的故障数量在给定时间段内超出了指定的阈值，则代理将被置于打开状态。此时，代理启动一个超时定时器，当该定时器到期时，代理被置于半打开状态。

> 超时定时器的目的是让系统有时间来解决导致故障的问题，然后允许应用程序再次执行该操作。

**打开**：应用程序的请求立即失败，返回异常给应用程序。

**半开**：允许来自应用程序的有限数量的请求通过并调用操作。如果这些请求成功，则假定以前导致故障的问题已经修复，断路器切换到关闭状态（故障计数器被复位）。如果任何请求失败，则断路器假定故障仍然存在，因此它恢复到打开状态并重新启动超时定时器，给系统更多的时间从故障中恢复。

> 半开状态有助于防止恢复服务突然被请求淹没。随着服务的恢复，它可能能够支持有限数量的请求，直到恢复完成，但是在恢复进行时，大量的工作可能导致服务超时或失败。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/circuit-breaker-diagram.png)

在图中，关闭状态使用的故障计数器是基于时间的。它会周期性的自动重置。这有助于防止断路器在偶然发生故障时进入打开状态。断路器跳闸到断开状态的故障阈值仅在指定时间间隔内发生指定数量的故障时达到。半开状态使用的计数器记录调用操作的成功尝试次数。在指定数量的连续操作调用成功后，断路器将恢复为“关闭”状态。如果任何调用失败，断路器立即进入打开状态，并且下一次进入半打开状态时，复位成功的计数器。

> 超时定时器的目的是让系统有时间来解决导致故障的问题，然后再允许应用程序重新执行该操作。

断路器模式提供稳定性，同时系统从故障中恢复，并最大限度地减少对性能的影响。它可以通过快速拒绝对可能失败的操作的请求来帮助维持系统的响应时间，而不是等待操作超时或永不返回。如果断路器在每次改变状态时引发事件，则该信息可用于监视由断路器保护的系统部分的健康状况，或者当断路器跳闸到打开状态时向管理员发出警报。

这种模式是可定制的，可以根据故障类型进行调整。例如，可以将断路器的定时器超时时间逐步增加。最初可以将断路器置于打开状态几秒钟，然后如果故障尚未解决会将超时增加到几分钟，依此类推。 在某些情况下，不是从打开状态返回失败并引发异常，返回对应用程序有意义的默认值可能更有用。

### 问题和注意事项

在决定如何实现此模式时，应该考虑以下几点：

**异常处理**。如果操作不可用，则必须准备通过断路器调用操作的应用程序来处理引发的异常。处理异常的方式将是具体的应用程序。例如，应用程序可能暂时将功能降级，调用替代操作来尝试执行相同的任务或获取相同的数据，或向用户报告异常，并请他们稍后再试。

**异常类型**。请求可能因为很多原因失败，其中一些可能表示比其它故障更严重的故障类型。例如，请求可能会失败，因为远程服务已崩溃，并且需要几分钟的时间才能恢复，或者因为服务暂时重载而导致超时。断路器可能会检查发生的异常类型，并根据这些异常的性质调整其策略。例如，与服务完全不可用导致的故障次数相比，可能需要更多的超时异常来将断路器跳闸到打开状态。

**日志记录**。断路器应记录所有失败的请求（可能成功的请求），以便管理员能够监视操作的健康状况。

**可恢复性**。应该配置断路器以匹配其保护操作的可能恢复模式。例如，如果断路器长时间保持在打开状态，即使解决了故障原因，也可能引起异常。类似地，如果能很快地从打开状态切换到半开状态，断路器可能会波动并降低应用的响应时间。

**测试失败的操作**。在打开状态下，断电器不使用定时器来确定何时切换到半开状态，而是可以定期ping远程服务或资源，以确定它是否再次可用。ping可以采取以前尝试调用先前失败的操作的形式，或者使用由远程服务提供的专门用于测试服务运行状况的特殊操作，如健康端点监控模式所述。

**手动覆盖**。在故障操作的恢复时间不确定的系统中，提供手动复位选项有助于管理员关闭断路器（并重置故障计数器）。类似地，如果由断路器保护的操作暂时不可用，管理员可以强制断路器进入打开状态（并重新启动超时定时器）。

**并发**。大量并发的应用程序可以访问相同的断路器。该实现不应阻止并发请求，或者每次调用操作会增加额外的开销。

**资源差异化**。对于一种类型的资源使用单个断路器时，如果存在多个潜在的独立提供者，请小心。 例如，在包含多个分片的数据存储中，一个分片可能完全可访问，而另一个分片遇到临时问题。如果在这些情况下的错误响应合并，应用程序可能尝试访问某些可能失败的分片，而访问其它可用的分片会被阻止。

**加速断路**。 有时，故障响应可以包含足够的信息，使断路器立即跳闸并保持最短时间的跳闸。 例如，来自重载的共享资源的错误响应可能表示不推荐立即重试，应该在几分钟之内重试。

> 注意：如果服务不可用，可以返回HTTP响应码429（太多请求），否则返回HTTP响应码503（服务不可用）。响应可以包括附加信息，例如延迟的预期持续时间。


**重放失败的请求**。 在打开状态下，断路器可以将每个请求的详细信息记录到日志中，而不是简单地快速失败，并在远程资源或服务可用时安排这些请求重放。

**外部服务不合适的超时**。断路器可能无法完全保护应用程序免受配置有超长超时时间的外部服务失败的操作。如果超时时间过长，在断路器指示操作失败之前，运行断路器的线程可能会长时间阻塞。这个时候，许多其它应用程序实例也可能尝试通过断路器调用该服务，并在它们全部失败之前绑定大量线程。

### 何时使用该模式

在以下场景使用该模式：

防止应用程序尝试调用远程服务或访问共享资源（如果此操作可能导致失败）。

以下场景不建议使用此模式：

* 处理应用程序中本地私有资源的访问，例如内存中数据结构。在这种环境中，使用断路器会增加系统的开销。
* 作为在应用程序的业务逻辑中处理异常的替代品。

### 例子

在Web应用程序中，从外部服务检索的数据会填充几个页面。如果系统实现最小的缓存，大多数页面的命中将导致服务的往返。从Web应用程序到服务的连接可以配置超时时间（通常为60秒），如果服务在这段时间内没有响应，则每个网页中的逻辑将假定该服务不可用并且引发异常。

但是，如果服务失败并且系统非常繁忙，用户可能会被迫最多等待60秒才能发现异常。最终，诸如内存，连接和线程之类的资源可能会耗尽，从而阻止其他用户连接到系统，即使他们没有访问从服务中检索数据的页面。

通过进一步的添加Web服务器和实现负载平衡来扩展系统可能会延迟资源耗尽的时间，但由于用户请求仍然无响应，最终所有Web服务器仍然可能用尽资源，问题依然没有解决。

将连接到服务和检索数据的逻辑包裹在断路器中可以帮助解决这个问题，同时更加优雅地处理服务故障。用户请求仍将失败，但是它们失败更快，因此资源将不会被阻塞。

`CircuitBreaker`类保存有关实现下面代码所示接口`ICircuitBreakerStateStore`的对象的断路器的状态信息。

```c#
interface ICircuitBreakerStateStore
{
  CircuitBreakerStateEnum State { get; }
  Exception LastException { get; }
  DateTime LastStateChangedDateUtc { get; }
  void Trip(Exception ex);
  void Reset();
  void HalfOpen();
  bool IsClosed { get; }
}
```

`State`属性表示断路器的当前状态，将由`CircuitBreakerStateEnum`枚举类定义断路器状态为打开，半开或闭合。断路器闭合时，`IsClosed`属性应为真，如果断路器为打开或半开。`Trip`方法将断路器的状态切换到打开状态，并记录导致状态更改的异常以及异常发生的日期和时间。`LastException`和`LastStateChangedDateUtc`属性返回此信息。`Reset`方法关闭断路器，`HalfOpen`方法将断路器设置为半开。

例子中的`InMemoryCircuitBreakerStateStore`类包含接口`ICircuitBreakerStateStore`的实现。`CircuitBreaker`类创建此类的实例以保持断路器的状态。

The ExecuteAction methodinthe classwrapsanoperation,specifiedasan Action delegate.If thecircuitbreakerisclosed, invokesthe Action delegate.Iftheoperationfails,anexception handler calls , which sets the circuit breaker state to open. The following code example highlights this flow.
`CircuitBreaker`类中的`ExecuteAction`方法将一个操作指定为`Action`代理。如果断路器关闭，`ExecuteAction`将调用`Action`代理。如果操作失败，则异常处理程序将调用`TrackException`，将断路器状态设置为打开。下面的代码展示了这个流程。

```c#
public class CircuitBreaker
{
  private readonly ICircuitBreakerStateStore stateStore =
    CircuitBreakerStateStoreFactory.GetCircuitBreakerStateStore();
  private readonly object halfOpenSyncObject = new object ();
  ...
  public bool IsClosed { get { return stateStore.IsClosed; } }
  public bool IsOpen { get { return !IsClosed; } }
  public void ExecuteAction(Action action)
  {
    ...
    if (IsOpen)
    {
      // The circuit breaker is Open.
      ... (see code sample below for details)
    }
    // The circuit breaker is Closed, execute the action.
    try
    {
      action(); 
    }
    catch (Exception ex)
    {
      // If an exception still occurs here, simply
      // retrip the breaker immediately.
      this.TrackException(ex);
      // Throw the exception so that the caller can tell
      // the type of exception that was thrown.
      throw;
    } 
 }
  private void TrackException(Exception ex)
  {
    // For simplicity in this example, open the circuit breaker on the first exception.
    // In reality this would be more complex. A certain type of exception, such as one
    // that indicates a service is offline, might trip the circuit breaker immediately.
    // Alternatively it might count exceptions locally or across multiple instances and
    // use this value over time, or the exception/success ratio based on the exception
    // types, to open the circuit breaker.
    this.stateStore.Trip(ex);
  }
}
```

下面的例子展示了断路器未关闭时执行的代码（上一个例子中省略掉的）。它首先检查断路器是否已经打开比`CircuitBreaker`类中本地`OpenToHalfOpenWaitTime`字段指定的时间长。如果是这种情况，`ExecuteAction`方法将断路器设置为半开，然后尝试执行由`Action`代理指定的操作。

如果操作成功，断路器将复位到关闭状态。如果操作失败，它将跳回到打开状态，并更新异常发生的时间，在再次执行操作之前，断路器将等待一段时间。

如果断路器只在短时间内打开，小于`OpenToHalfOpenWaitTime`的值，则`ExecuteAction`方法会抛出`CircuitBreakerOpenException`异常并返回导致断路器转换到打开状态的错误。

此外，它使用锁来防止断路器在半开状态下尝试对该操作执行并发调用。尝试并发调用该操作将像断路器打开一样被处理，如下所述，它将失败并且抛出异常。

```java
...
if (IsOpen)
{
  // The circuit breaker is Open. Check if the Open timeout has expired.
  // If it has, set the state to HalfOpen. Another approach might be to
  // check for the HalfOpen state that had be set by some other operation.
  if (stateStore.LastStateChangedDateUtc + OpenToHalfOpenWaitTime < DateTime.UtcNow)
  {
    // The Open timeout has expired. Allow one operation to execute. Note that, in
    // this example, the circuit breaker is set to HalfOpen after being
    // in the Open state for some period of time. An alternative would be to set
    // this using some other approach such as a timer, test method, manually, and
    // so on, and check the state here to determine how to handle execution
    // of the action.
    // Limit the number of threads to be executed when the breaker is HalfOpen.
    // An alternative would be to use a more complex approach to determine which
    // threads or how many are allowed to execute, or to execute a simple test
    // method instead.
    bool lockTaken = false;
    try
    {
      Monitor.TryEnter(halfOpenSyncObject, ref lockTaken)
      if (lockTaken)
      {
        // Set the circuit breaker state to HalfOpen.
        stateStore.HalfOpen();
        // Attempt the operation.
        action();
        // If this action succeeds, reset the state and allow other operations.
        // In reality, instead of immediately returning to the Closed state, a counter
        // here would record the number of successful operations and return the
        // circuit breaker to the Closed state only after a specified number succeed.
        this.stateStore.Reset();
        return;
      }
      catch (Exception ex)
      {
        // If there's still an exception, trip the breaker again immediately.
        this.stateStore.Trip(ex);
        // Throw the exception so that the caller knows which exception occurred.
        throw; 
       }
      finally {
        if (lockTaken)
        {
          Monitor.Exit(halfOpenSyncObject);
        }
      } 
    }
  }
  // The Open timeout hasn't yet expired. Throw a CircuitBreakerOpen exception to
  // inform the caller that the call was not actually attempted,
  // and return the most recent exception received.
  throw new CircuitBreakerOpenException(stateStore.LastException);
} ...

```

Tousea objecttoprotectanoperation,anapplicationcreatesaninstanceofthe CircuitBreaker class and invokes the method, specifying the operation to be performed as the parameter. The
CircuitBreaker
 ExecuteAction
application should be prepared to catch the exception if the operation fails because the circuit breaker is open. The following code shows an example:

要使用`CircuitBreaker`对象来保护操作，应用程序将创建`CircuitBreaker`类的实例，并调用`ExecuteAction`方法，指定要作为参数执行的操作。如果由于断路器断开而导致操作失败，应用程序应准备捕获`CircuitBreakerOpenException`异常。下面为示例代码：

```c#
var breaker = new CircuitBreaker();
try {
  breaker.ExecuteAction(() =>
  {
    // Operation protected by the circuit breaker.
    ... 
  });
}
catch (CircuitBreakerOpenException ex)
{
  // Perform some different action when the breaker is open.
  // Last exception details are in the inner exception.
  ...
}
catch (Exception ex)
{
  ... 
}
```

### 相关模式和指导

实现此模式时，以下模式也可能很有用：

* [重试模式](retry.html)。描述当应用程序通过透明地重试以前失败的操作尝试连接到服务或网络资源时，应用程序如何处理预期的临时故障。
* [健康端点监控模式](health-endpoint-monitoring.html)。断路器可能能够通过向服务器公开的端点发送请求来测试服务的健康状况。 服务应返回指示其状态的信息。