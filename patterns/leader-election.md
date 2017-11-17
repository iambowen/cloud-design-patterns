# 领导者模式
通过选择一个实例作为管理其它实例的领导者，协调分布式应用程序中的一系列协作实例所执行的操作。 这可以确保实例不会因为相互冲突，导致争用共享资源，或者无意中干扰其它实例正在执行的工作。

## 背景和问题
典型的云应用程序以协调的方式运行许多任务。这些任务可以是运行相同代码并访问相同资源的实例，也可以并行执行复杂计算的各个部分。
任务实例大部分时间可能在单独运行，也可能需要协调每个实例的操作，以确保不会发生冲突，导致共享资源的争用，或者意外地干扰其它正在运行的任务实例的工作。
例如：
* 在实现水平缩放的基于云的系统中，可以同时运行同一任务的多个实例，每个实例服务于不同的用户。如果这些实例写入共享资源，则必须协调其操作以防止每个实例覆盖其它实例所做的更改。
* 如果任务并行执行复杂计算的各个元素，则在完成所有结果时需要汇总结果。
任务实例都是对等的，所以没有一个自然的领导可以充当协调者或聚合者。

## 解决方案

应该选择一个任务实例作为领导者，用来协调其它从属任务实例的动作。如果所有任务实例都运行相同的代码，那么它们每个都可以充当领导者。因此，必须认真管理选举过程，防止两个或更多的实例同时充当领导者角色。
系统必须提供一个强大的选择领导者的机制。这种方法必须处理诸如网络中断或流程失败等事件。在许多解决方案中，下级任务实例通过某种类型的心跳方法或通过轮询来监视领导者。如果指定的领导者意外终止，或者网络故障使得领导者不能从属于任务实例，那么有必要选出一个新的领导者。
在分布式环境中的一组任务中选择领导者有几种策略，包括：
* 选择排名最低的实例或进程ID的任务实例。
* 竞赛以获得共享的，分布式的互斥量。获得互斥量的第一个任务实例是领导者。然而，系统必须确保领导者终止或与系统的其它部分断开连接时，释放互斥体以允许另一个任务实例成为领导者。
* 实现常见的领导者选举算法之一，比如Bully算法或者Ring算法。这些算法假定选举中的每个候选人都有一个唯一的ID，并且可以可靠地与其它候选人通信。

## 问题和注意事项
在决定如何实现该模式时，请考虑以下几点：
* 选举领导者的过程应该是对瞬态和持续失败具备弹性。
* 必须能够检测领导者何时失败或不可用（例如由于通信故障）。如何快速检测是系统相关的。有些系统可能在没有领导者的情况下短时间工作，在此期间解决瞬时故障。在其它情况下，可能需要立即检测领导者失败并引发新的选举。
* 在实现水平自动缩放的系统中，如果系统收缩并关闭一些计算资源，则可以终止领导者。
* 使用共享的分布式互斥量引入了对提供互斥量的外部服务的依赖。该服务构成单点故障。如果因任何原因无法使用，系统将无法选举领导者。
* 使用专门的流程作为领导是一个直接的方法。但是，如果进程失败，那么在重新启动时可能会有明显的延迟。如果等待领导者协调一个操作，则所产生的延迟会影响其它进程的性能和响应时间。
* 实现一个领导者选举算法手动提供了最大的灵活性来调整和优化代码。

## 何时使用该模式
当分布式应用程序中的任务（如云托管解决方案）需要仔细协调并且没有天生的领导者时，请使用此模式：
* 避免让领导者成为系统中的瓶颈。领导者的目的是协调下属任务的工作，而不一定要参与这项工作本身-尽管如果任务不被指派给领导者就可以了。
以下情况可能不适合该模式：
* 有一个自然的领导者或专门的过程，总是可以扮演领导者。例如，可以实现协调任务实例的单例过程。如果此过程失败或变得不健康，系统可以关闭并重新启动。
* 可以使用更轻量级的方法来实现任务之间的协调。例如，如果几个任务实例只需要对共享资源进行协调访问，则更好的解决方案是使用乐观或悲观锁定来控制访问。
* 第三方解决方案更合适。例如，Microsoft Azure HDInsight服务（基于Apache Hadoop）使用Apache Zookeeper提供的服务来协调收集和汇总数据的map reduce任务。

## 案例
`LeaderElection`解决方案中的`DistributedMutex`项目（此模式的演示代码可以[GitHub](https://github.com/mspnp/cloud-design-patterns/tree/master/leader-election)上找到）显示了如何使用Azure存储区域上的租约来提供实现共享的分布式互斥量的机制。这个互斥量可以用来选择Azure云服务中的一组角色实例中的领导者。获得租赁的第一个角色实例被选举为领导者，直到它解除租约或不能续租为止。其他角色实例可以继续监视blob租约，以防领导者不再可用。

>blob租约是blob上的独占写入锁。在任何时间点，一个blob可能只是一个租约的主题。角色实例可以通过指定的blob请求租约，如果没有其它角色实例在同一个blob上持有租约，它将被授予租约。否则，请求会抛出异常。
>为避免故障的角色实例无限期地保留租约，请为租约指定一个生命周期。租约到期时将变为可用。但是，角色实例持有租约时，可以要求续租，并且将续租一段时间。角色实例可以不断重复这个过程，如果它想保留租约。有关如何租用Blob的更多信息，请参阅[租赁Blob（REST API）](https://msdn.microsoft.com/library/azure/ee691972.aspx)。

下面的C＃示例中的`BlobDistributedMutex`类包含的`RunTaskWhenMutexAquired`方法允许角色实例尝试获取指定blob上的租约。创建`BlobDistributedMutex`对象（该对象是包含在示例代码中的简单结构体）时，Blob（名称，容器和存储帐户）的详细信息将传递给`BlobSettings`对象中的构造函数。构造函数还接受一个Task，该Task引用角色实例应该运行的代码，如果它成功地获得blob上的租约并成为领导者。请注意，处理获取租约的底层详细代码是在名为`BlobLeaseManager`的单独的辅助类中实现的。

```c#
public class BlobDistributedMutex
{
  ...
  private readonly BlobSettings blobSettings;
  private readonly Func<CancellationToken, Task> taskToRunWhenLeaseAcquired;
  ...

  public BlobDistributedMutex(BlobSettings blobSettings,
           Func<CancellationToken, Task> taskToRunWhenLeaseAquired)
  {
    this.blobSettings = blobSettings;
    this.taskToRunWhenLeaseAquired = taskToRunWhenLeaseAquired;
  }

  public async Task RunTaskWhenMutexAcquired(CancellationToken token)
  {
    var leaseManager = new BlobLeaseManager(blobSettings);
    await this.RunTaskWhenBlobLeaseAcquired(leaseManager, token);
  }
  ...
```
上面代码中的`RunTaskWhenMutexAquired`方法调用以下代码中的`RunTaskWhenBlobLeaseAcquired`方法以获取实际租约。 `RunTaskWhenBlobLeaseAcquired`方法异步运行。如果成功获得租约，角色实例将成为领导者。`taskToRunWhenLeaseAcquired`委托的目的是执行协调其他角色实例的工作。 如果没有获得租约，则选择另一个角色实例作为领导者，而当前角色实例仍然是下属。请注意，`TryAcquireLeaseOrWait`方法是使用`BlobLeaseManager`对象获取租约的帮助方法。
```c#
  private async Task RunTaskWhenBlobLeaseAcquired(
    BlobLeaseManager leaseManager, CancellationToken token)
  {
    while (!token.IsCancellationRequested)
    {
      // Try to acquire the blob lease.
      // Otherwise wait for a short time before trying again.
      string leaseId = await this.TryAquireLeaseOrWait(leaseManager, token);

      if (!string.IsNullOrEmpty(leaseId))
      {
        // Create a new linked cancellation token source so that if either the
        // original token is canceled or the lease can't be renewed, the
        // leader task can be canceled.
        using (var leaseCts =
          CancellationTokenSource.CreateLinkedTokenSource(new[] { token }))
        {
          // Run the leader task.
          var leaderTask = this.taskToRunWhenLeaseAquired.Invoke(leaseCts.Token);
          ...
        }
      }
    }
    ...
  }
```
领导者开始的任务也是异步运行的。在任务正在运行时，以下代码中的`RunTaskWhenBlobLeaseAquired`方法会定期尝试续订租约。 这有助于确保角色实例仍然是领导者。在案例解决方案中，续订请求之间的延迟时间小于租用期间指定的时间，以防止另一个角色实例被选为领导者。 如果由于任何原因续约失败，取消任务。
如果租约未能续订或任务取消（可能因为角色实例关闭），则租约将被释放。此时，这个或另一个角色实例可能被选为领导者。下面的代码断显示了这部分过程。
```c#
  private async Task RunTaskWhenBlobLeaseAcquired(
    BlobLeaseManager leaseManager, CancellationToken token)
  {
    while (...)
    {
      ...
      if (...)
      {
        ...
        using (var leaseCts = ...)
        {
          ...
          // Keep renewing the lease in regular intervals.
          // If the lease can't be renewed, then the task completes.
          var renewLeaseTask =
            this.KeepRenewingLease(leaseManager, leaseId, leaseCts.Token);

          // When any task completes (either the leader task itself or when it
          // couldn't renew the lease) then cancel the other task.
          await CancelAllWhenAnyCompletes(leaderTask, renewLeaseTask, leaseCts);
        }
      }
    }
  }
  ...
}
```
`KeepRenewingLease`方法是使用`BlobLeaseManager`对象更新租约的另一个辅助方法。`CancelAllWhenAnyCompletes`方法取消指定为前两个参数的任务。下图说明了如何使用`BlobDistributedMutex`类来选举领导者并运行协调操作的任务。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/leader-election-diagram.png)
以下代码示例展示了如何在辅助角色中使用`BlobDistributedMutex`类。代码通过开发存储中的租约容器中名为`MyLeaderCoordinatorTask`的blob获取租约，如果角色实例被选为领导者，则运行`MyLeaderCoordinatorTask`方法中定义的代码。
```c#
var settings = new BlobSettings(CloudStorageAccount.DevelopmentStorageAccount,
  "leases", "MyLeaderCoordinatorTask");
var cts = new CancellationTokenSource();
var mutex = new BlobDistributedMutex(settings, MyLeaderCoordinatorTask);
mutex.RunTaskWhenMutexAcquired(this.cts.Token);
...

// Method that runs if the role instance is elected the leader
private static async Task MyLeaderCoordinatorTask(CancellationToken token)
{
  ...
}
```
请注意关于案例解决方案中以下几点：
* blob存在单点故障的可能。如果blob服务变得不可用或无法访问，领导者将不能续订租约，并且其他任何角色实例都不能获得租约。在这种情况下，任何角色实例都不能成为领导者。然而，blob服务被设计为具有弹性，所以blob服务的完全失败被不太可能。
* 如果领导者执行的任务失败，可能会继续更新租约，阻止任何其他角色实例获得租约并接管领导角色以协调任务。在现实世界中，应该经常检查领导者的健康状况。
* 选举过程是不确定的。不能假定哪个角色实例会获得blob租约并成为领导者。
* 用作blob租赁目标的blob不应该用于其它任何目的。如果角色实例试图将数据存储在此blob中，则除非角色实例是领导者并拥有blob租约，否则将无法访问此数据。

## 相关模式和指南

以下模式和指南在实现此模式时可能相关：
* 这种模式有一个可示例应用程序可以在[这里](https://github.com/mspnp/cloud-design-patterns/tree/master/leader-election)下载。
* [自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。应用程序的负载变化时，可以启动和停止任务主机的实例。自动缩放可以在高峰期间帮助保持吞吐量和性能。
* [计算分区指南](https://msdn.microsoft.com/library/dn589773.aspx)。该指南介绍了如何以一种有助于最大限度降低运营成本同时保持服务的可伸缩性，性能，可用性和安全性的方式，将任务分配给云服务中的主机。
* [基于任务的异步模式](https://msdn.microsoft.com/library/hh873175.aspx)。
* [介绍欺负算法的例子](http://www.cs.colostate.edu/~cs551/CourseNotes/Synchronization/BullyExample.html)。
* [介绍环算法的例子](http://www.cs.colostate.edu/~cs551/CourseNotes/Synchronization/RingElectExample.html)。
* Microsoft Open Technologies网站上关于[Apache Zookeeper在Microsoft Azure上使用的文章](https://msopentech.com/opentech-projects/apache-zookeeper-on-windows-azure-2/)。
* [Apache Curator](http://curator.apache.org/)是Apache ZooKeeper的客户端库。
* MSDN上的[Lease Blob（REST API）文章](https://msdn.microsoft.com/library/azure/ee691972.aspx)。