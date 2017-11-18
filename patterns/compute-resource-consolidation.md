# 计算资源整合模式

将多个任务或操作合并到一个计算单元中。这可以提高计算资源利用率，并降低与在云托管应用程序中执行计算处理相关的成本和管理开销。

## 背景和问题

云应用程序经常实现各种操作。在一些解决方案中，遵循最初关注分离的设计原则是有意义的，并将这些操作划分为单独的计算单元，这些计算单元分别托管和部署（例如，作为单独的App Service Web应用程序，单独的虚拟机或单独的Cloud Service角色）。但是，尽管这种策略可以帮助简化解决方案的逻辑设计，但将大量计算单元作为同一应用程序的一部分来部署会增加运行时托管成本，并使系统管理更为复杂。

例如，下图显示了使用多个计算单元实现的云托管解决方案的简化结构。每个计算单元都运行在自己的虚拟环境中。每个功能都是作为一个单独的任务来实现的（标记为任务A到任务E）以自己的计算单位运行。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/compute-resource-consolidation-diagram.png)

每个计算单位即使闲置或轻微使用，也会消耗收费的资源。因此，这并不总是最具成本效益的解决方案。
这种担心适用于Azure的Cloud Service，App Services和虚拟机中的角色。这些项目在自己的虚拟环境中运行。运行一组单独的角色，网站或虚拟机，这些角色，网站或虚拟机旨在执行一组明确定义的操作，但需要作为单一解决方案的一部分进行通信和合作，可能会导致资源的低效使用。

## 解决方案

为了帮助降低成本，提高利用率，提高通信速度并减少管理，可以将多个任务或操作合并为一个计算单元。

可以根据环境提供的功能和与这些功能相关的成本，将任务分组。一种常见的方法是查找具有与其可扩展性，生命周期和处理要求相似的任务。组合在一起使它们成为一个整体。许多云环境提供的弹性使计算单元的其它实例可以根据工作负载来启动和停止。例如，Azure提供了自动伸缩，可以将其应用于Cloud Service，App Services和虚拟机中的角色。更多相关内容，请参阅[自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。

作为一个反例来说明如何使用可伸缩性来确定哪些操作不应该组合在一起，请考虑以下两个任务：

   *  任务1轮询发送到队列的偶发的，时间不敏感的消息。
   *  任务2处理大量的网络流量突发。

第二个任务需要弹性，可能涉及到启动和停止计算单元的大量实例。对第一个任务应用相同的伸缩比例只会导致更多的任务在同一队列上侦听不频繁的消息，并且浪费资源。

在许多云环境中，可以根据CPU核心数量，内存，磁盘空间等指定可用于计算单元的资源。一般来说，指定的资源越多，成本越高。为了节省资金，重要的是要使昂贵的计算单元执行的工作最大化，并且不要让它在很长一段时间内变为非活动状态。
如果有些任务在短时间内需要大量的CPU，请考虑将这些任务合并到一个提供必要能力的计算单元中。然而，需要在昂贵的资源保持忙碌和让它们过载之间保持平衡是非常重要的。例如，长时间运行的计算密集型任务不应共享相同的计算单位。

## 问题和注意事项

实施此模式时请考虑以下几点：
* **可伸缩性和弹性**。许多云计算解决方案通过启动和停止单元实例来实现计算单元级别的可伸缩性和弹性。避免在相同的计算单元中对具有可伸缩性要求冲突的任务进行分组。
* **生命周期**。云基础架构定期回收承载计算单元的虚拟环境。当一个计算单元内有很多长时间运行的任务时，可能需要配置该单元以防止它在这些任务完成之前被回收。或者，通过使用检查点方法设计任务，使其能够干净地停止，并在计算单元重新启动时在中断处继续。
* **发布节奏**。如果任务的实现或配置频繁更改，则可能需要停止承载更新代码的计算单元，重新配置和部署单元，然后重新启动它。这个过程还要求停止，重新部署和重新启动同一计算单元内的所有其它任务。
* **安全**。同一计算单元中的任务可能共享相同的安全上下文并访问相同的资源。任务之间必须有高度的信任，相信一项任务不会对另一项任务产生不利影响。此外，增加计算单元中运行的任务数量会增加单元的攻击面。每个任务只与具有最多漏洞的任务一样安全。
* **容错**。如果计算单元中的一个任务失败或行为异常，则可能影响同一单元内运行的其它任务。例如，如果某个任务无法正确启动，则可能导致计算单元的整个启动逻辑失败，并阻止同一单元中的其它任务运行。
* **竞争**。避免在同一计算单元中争夺资源的任务之间引入竞争。理想情况下，共享相同计算单元的任务应该表现出不同的资源利用特性。例如，两个计算密集型任务可能不应该驻留在相同的计算单元中，也不应该占用大量内存的两个任务也不应该在一起。但是，将计算密集型任务与需要大量内存的任务混合在一起是可行的组合。
>注意
>考虑整合计算资源只能用于一段时间内已经投入生产的系统，这样运维和开发人员才能监视系统并创建一个热图来标识每个任务如何利用不同的资源。该图可用于确定哪些任务是共享计算资源的合适候选者。
* **复杂性**。将多个任务合并到一个单一的计算单元会增加单元中代码的复杂性，可能使测试，调试和维护变得更加困难。
* **稳定的逻辑架构**。在每个任务中设计并实现代码，使其不需要更改，即使任务运行的物理环境确实发生了变化。
* **其他策略**。整合计算资源只是帮助降低与同时运行多个任务相关的成本的一种方法。这需要仔细的计划和监督，以确保它仍然是一个有效的方法。其它策略可能更合适，这取决于工作的性质以及这些任务运行的用户所在的位置。例如，对工作负载的功能分解（如[计算分区指南](https://msdn.microsoft.com/library/dn589773.aspx)所述）可能是更好的选择。

## 何时使用该模式

如果这些模式以自己的计算单位运行，那么使用这种模式性价比不高。如果任务大部分时间闲置，那么在专用单元中执行此任务可能会很昂贵。
这种模式可能不适用于执行关键容错操作的任务，或者处理高度敏感或私有数据并需要其自身安全上下文的任务。这些任务应该在独立的环境中运行，在一个单独的计算单元中运行。

## 案例

在Azure上构建云服务时，可以将多个任务执行的处理合并为一个角色。通常这是执行后台或异步处理任务的工作者角色。

> 在某些情况下，可以在Web角色中包含后台或异步处理任务。这种技术有助于降低成本并简化部署，但它会影响Web角色提供的面向公众的接口的可伸缩性和响应能力。[将多个Azure角色组合到Azure Web角色中](http://www.31a2ba2a-b718-11dc-8314-0800200c9a66.com/2012/02/combining-multiple-azure-worker-roles.html)的文章包含了在Web角色中实现后台或异步处理任务的详细说明。

角色负责启动和停止任务。当Azure结构控制器加载角色时，会引发该角色的“启动”事件。可以重写`WebRole`或`WorkerRole`类的`OnStart`方法来处理此事件，也许可以初始化此方法中任务所依赖的数据和其它资源。

当`OnStartmethod`完成时，角色可以开始响应请求。可以在模式与实践指南中的[“应用程序启动过程”](https://msdn.microsoft.com/library/ff728592.aspx)部分找到有关在角色中使用`OnStart`和`Run`方法的更多内容。

>保持OnStart方法中的代码尽可能简洁。 Azure对此方法完成所花费的时间没有任何限制，但在此方法完成之前，角色将无法响应发送给它的网络请求。

当`OnStart`方法完成时，角色执行`Run`方法。此时，结构控制器可以开始向角色发送请求。
将实际创建任务的代码放在`Run`方法中。请注意，`Run`方法定义了角色实例的生命周期。当此方法完成时，结构控制器将安排角色关闭。
当角色关闭或回收时，结构控制器会阻止从负载均衡器接收到更多传入请求，并引发`Stop`事件。可以通过重写角色的`OnStop`方法来捕获此事件，并在角色终止之前执行所需的任何整理。

>在`OnStop`方法中执行的任何操作都必须在五分钟内完成（如果在本地计算机上使用Azure模拟器，则必须等待30秒）。否则，Azure结构控制器将假定角色已停止并将强制停止。

Run方法启动任务并等待任务完成的。这些任务实现了云服务的业务逻辑，并可以通过Azure负载均衡器响应发布到角色的消息。下图显示了Azure云服务中角色中任务和资源的生命周期。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/compute-resource-consolidation-lifecycle.png)

`ComputeResourceConsolidation.Worker`项目中的`WorkerRole.cs`文件展示了一个例子，说明如何在Azure云服务中实现此模式。

>`ComputeResourceConsolidation.Worker`项目是`ComputeResourceConsolidation`解决方案的一部分，可以从[GitHub下载](https://github.com/mspnp/cloud-design-patterns/tree/master/compute-resource-consolidation)。

`MyWorkerTask1`和`MyWorkerTask2`方法说明了如何在同一个辅助角色中执行不同的任务。以下代码展示了`MyWorkerTask1`。这是一个简单的任务，休眠30秒，然后输出一个跟踪消息。它重复这个过程直到任务取消。 `MyWorkerTask2`的代码和这个类似。

```C#

// A sample worker role task.
private static async Task MyWorkerTask1(CancellationToken ct)
{
  // Fixed interval to wake up and check for work and/or do work.
  var interval = TimeSpan.FromSeconds(30);

  try
  {
    while (!ct.IsCancellationRequested)
    {
      // Wake up and do some background processing if not canceled.
      // TASK PROCESSING CODE HERE
      Trace.TraceInformation("Doing Worker Task 1 Work");

      // Go back to sleep for a period of time unless asked to cancel.
      // Task.Delay will throw an OperationCanceledException when canceled.
      await Task.Delay(interval, ct);
    }
  }
  catch (OperationCanceledException)
  {
    // Expect this exception to be thrown in normal circumstances or check
    // the cancellation token. If the role instances are shutting down, a
    // cancellation request will be signaled.
    Trace.TraceInformation("Stopping service, cancellation requested");

    // Rethrow the exception.
    throw;
  }
}
```
>示例代码显示了后台进程的常见实现。在真实世界的应用程序中，可以遵循相同的结构，只是应该将处理逻辑放在等待取消请求的循环体中。

在worker角色初始化所使用的资源之后，Run方法将同时启动两个任务，如下所示。
```C#

/// <summary>
/// The cancellation token source use to cooperatively cancel running tasks
/// </summary>
private readonly CancellationTokenSource cts = new CancellationTokenSource();

/// <summary>
/// List of running tasks on the role instance
/// </summary>
private readonly List<Task> tasks = new List<Task>();

// RoleEntry Run() is called after OnStart().
// Returning from Run() will cause a role instance to recycle.
public override void Run()
{
  // Start worker tasks and add to the task list
  tasks.Add(MyWorkerTask1(cts.Token));
  tasks.Add(MyWorkerTask2(cts.Token));

  foreach (var worker in this.workerTasks)
  {
      this.tasks.Add(worker);
  }

  Trace.TraceInformation("Worker host tasks started");
  // The assumption is that all tasks should remain running and not return,
  // similar to role entry Run() behavior.
  try
  {
    Task.WaitAll(tasks.ToArray());
  }
  catch (AggregateException ex)
  {
    Trace.TraceError(ex.Message);

    // If any of the inner exceptions in the aggregate exception
    // are not cancellation exceptions then re-throw the exception.
    ex.Handle(innerEx => (innerEx is OperationCanceledException));
  }

  // If there wasn't a cancellation request, stop all tasks and return from Run()
  // An alternative to canceling and returning when a task exits would be to
  // restart the task.
  if (!cts.IsCancellationRequested)
  {
    Trace.TraceInformation("Task returned without cancellation request");
    Stop(TimeSpan.FromMinutes(5));
  }
}
...
```
在这个例子中，Run方法等待任务完成。如果任务取消，则Run方法假定角色正在关闭，并在完成之前等待剩余的任务取消（终止之前最多等待5分钟）。如果某个任务由于预期的异常而失败，则Run方法将取消该任务。

>可以在Run方法中实施更全面的监视和异常处理策略，例如重启失败的任务，或包含使角色停止和启动单个任务的代码。

当结构控制器关闭角色实例（从`OnStop`方法调用的）时，将调用以下代码中的`Stop`方法。代码通过取消它来优雅地停止每个任务。 如果任何任务完成要超过五分钟，则Stop方法中的取消处理停止等待并且终止角色。

```C#
// Stop running tasks and wait for tasks to complete before returning
// unless the timeout expires.
private void Stop(TimeSpan timeout)
{
  Trace.TraceInformation("Stop called. Canceling tasks.");
  // Cancel running tasks.
  cts.Cancel();

  Trace.TraceInformation("Waiting for canceled tasks to finish and return");

  // Wait for all the tasks to complete before returning. Note that the
  // emulator currently allows 30 seconds and Azure allows five
  // minutes for processing to complete.
  try
  {
    Task.WaitAll(tasks.ToArray(), timeout);
  }
  catch (AggregateException ex)
  {
    Trace.TraceError(ex.Message);

    // If any of the inner exceptions in the aggregate exception
    // are not cancellation exceptions then rethrow the exception.
    ex.Handle(innerEx => (innerEx is OperationCanceledException));
  }
}
```

## 相关的模式和指南

以下模式和指南在实施这种模式时可能是相关的：

* [自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。自动缩放可用于启动和停止服务托管计算资源的实例，具体取决于预期的处理需求。
* [计算分区指南](https://msdn.microsoft.com/library/dn589773.aspx)。本文介绍了如何在云服务中分配服务和组件，以便在保持服务的可伸缩性，性能，可用性和安全性的同时最大限度降低运营成本。
* 该模式的一个[示例应用程序](https://github.com/mspnp/cloud-design-patterns/tree/master/compute-resource-consolidation)。