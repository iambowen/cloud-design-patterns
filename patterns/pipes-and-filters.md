# 管道和过滤器模式

将执行复杂处理的任务分解为一系列可重用的单独元素。通过允许执行处理的任务元素独立部署和扩展来提高性能，可伸缩性和可重用性。

## 背景和问题

应用程序需要在处理信息时执行各种复杂性不同的任务。直接但是不灵活的实现应用程序的方式是采用单体模块执行处理。如果在应用程序的其它地方需要部分相同的处理，这种方式可能会减少重构，优化或重用的机会。
下图说明了使用单体方式处理数据的问题。应用程序接收和处理的数据有两个来源。每个来源的数据由一个单独的模块处理，该模块在将结果传递给应用程序的业务逻辑之前执行一系列的任务来转换数据。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/pipes-and-filters-modules.png)

单体模块执行的一些任务在功能上非常相似，但模块是分开设计的。实现这些任务的代码耦合在一个模块中，并且在很少或根本没有考虑重用或可伸缩性的情况下开发。
但是，每个模块执行的处理任务或每个任务的部署要求可能随着业务需求的更新而改变。一些任务可能是计算密集型的，可以在强大的硬件上运行，而其它任务可能不需要这样昂贵的资源。另外，将来可能需要额外的处理，或者处理所执行的任务的顺序可能会改变。需要解决这些问题的解决方案，并增加了代码重用的可能性。

## 解决方案

将每个流所需的处理分解成一组单独的组件（或过滤器），每个组件执行单个任务。通过标准化每个组件接收和发送的数据的格式，这些过滤器可以组合在一起成为一个流水线。这有助于避免重复代码，并且在处理需求发生变化时可以轻松地删除，替换或集成其它组件。下图展示了使用管道和过滤器实现的解决方案。

![https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/pipes-and-filters-solution.png]

处理单个请求所花费的时间取决于流水线中最慢的过滤器的速度。一个或多个过滤器可能是瓶颈，特别是如果大量请求出现在来自特定数据源的流中。流水线结构的一个关键优势是它为慢速过滤器的并行运行提供了机会，分散系统负载并提高吞吐量。

组成管道的过滤器可以运行在不同的机器上，使其可以独立扩展，并充分利用许多云环境提供的弹性。计算密集型的过滤器可以在高性能硬件上运行，而其他要求不高的过滤器则可以在更便宜的商品硬件上运行。过滤器甚至不必位于相同的数据中心或地理位置，这使得管道中的每个元素都可以在接近所需资源的环境中运行。下图显示了应用于来自Source 1的数据的管道的示例。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/pipes-and-filters-load-balancing.png)
如果过滤器输入和输出以流的方式构造，那么每个滤波器就可以并行执行处理。流水线中的第一个过滤器可以开始工作并输出结果，在第一个过滤器完成工作之前，结果直接顺序的传递下一个过滤器。
另一个好处是这个模型可以提供弹性。如果过滤器发生故障或者运行的机器不再可用，则管道可以重新安排过滤器正在执行的工作，并将此工作指向组件的另一个实例。单个过滤器的故障不一定会导致整个管道的故障。
将管道和过滤器模式与补偿交易模式结合使用是另一种实现分布式事务的方法。分布式事务可以分解为单独的，可补偿的任务，每个任务都可以通过使用实现补偿事务模式的过滤器来实现。流水线中的过滤器可以作为独立的托管任务，在靠近它们维护的数据的地方运行。

### 问题和注意事项

在决定如何实现这种模式时应该考虑以下几点：
* **复杂性**。这种模式增加了灵活性但同时也带来复杂性，特别是如果管道中的过滤器分布在不同的服务器上。
* **可靠性**。使用可确保流水线中的过滤器之间的数据流不会丢失的基础设施。
* **幂等**。如果管道中的过滤器在收到消息后失败，并且重新安排工作到过滤器的另一个实例，则部分工作可能已经完成。如果这项工作更新了全局状态的某些方面（例如存储在数据库中的信息），则可以重复相同的更新。如果过滤器在将结果发布到管道中的下一个过滤器之后但在指示过程成功完成之前失败，可能会发生类似的问题。在这些情况下，相同的工作可以通过过滤器的另一个实例重复，导致相同的结果发布两次。可能导致流水线中的后续过滤器重复处理相同的数据。因此，管道中的过滤器应设计为幂等。更多相关内容，请参阅Jonathan Oliver的博客上的[幂等模式](http://blog.jonathanoliver.com/idempotency-patterns/)。
* **重复消息**。如果管道中的过滤器在将消息发布到管道的下一个阶段后失败，则可能会运行另一个过滤器实例，将相同消息的副本发布到管道。这可能导致相同消息的两个实例传递到下一个过滤器。为了避免这种情况，管道应检测并消除重复的消息。
> 如果使用消息队列（如Microsoft Azure服务总线队列）来实现管道，则消息队列基础结构可能会提供自动重复消息检测和删除功能。
* **上下文和状态**。在一个流水线中，每个过滤器本质上都是独立运行的，不应该假设如何调用它。这意味着每个过滤器都应有足够的上下文来执行其工作。这个上下文可能包含大量的状态信息。

## 何时使用该模式

在以下场景使用此模式：
* 应用程序所需的处理可以很容易地分解为一系列独立的步骤。
* 应用程序执行的处理步骤具有不同的可伸缩性要求。
>可以在同一个过程（进程？）中将过滤器分组在一起伸缩。更多相关内容，请参阅[计算资源合并模式](https://docs.microsoft.com/en-us/azure/architecture/patterns/compute-resource-consolidation)。
* 需要重新排序由应用程序执行的处理步骤的灵活性，或者添加和移除步骤的能力。
* 系统可以通过分散跨不同服务器的步骤的处理而受益。
* 需要一个可靠的解决方案，以最大限度地减少数据正在处理中的一个步骤中的故障造成的影响。

此模式可能不适用一下场景：
* 应用程序执行的处理步骤不是独立的，或者它们必须作为同一事务的一部分一起执行。
* 步骤所需的上下文或状态信息量使得这种方法效率低下。可以将状态信息持久化到数据库，但是如果数据库上的额外负载导致过度争用，则不要使用此策略。

## 案例
可以使用一系列消息队列来提供实现管道所需的基础架构。初始消息队列接收未处理的消息。做为过滤器任务的组件在此队列上侦听消息，执行工作，然后将已转换的消息发布到序列中的下一个队列中。另一个过滤器任务可以侦听并处理此队列上的消息，将结果发布到另一个队列中，依此类推，直到完全转换的数据出现在队列中的最终消息中。下图说明了如何使用消息队列来实现流水线。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/pipes-and-filters-message-queues.png)
如果在Azure上构建解决方案，则可以使用`Service Bus`队列服务来提供可靠且可扩展的排队机制。下面的C＃代码中的`ServiceBusPipeFilter`类展示了如何实现从队列接收输入消息，处理消息并将结果发布到另一个队列的过滤器。
> `ServiceBusPipeFilter`类在GitHub上的[PipesAndFilters.Shared项目](https://github.com/mspnp/cloud-design-patterns/tree/master/pipes-and-filters)中找到。

```c#
public class ServiceBusPipeFilter
{
  ...
  private readonly string inQueuePath;
  private readonly string outQueuePath;
  ...
  private QueueClient inQueue;
  private QueueClient outQueue;
  ...

  public ServiceBusPipeFilter(..., string inQueuePath, string outQueuePath = null)
  {
     ...
     this.inQueuePath = inQueuePath;
     this.outQueuePath = outQueuePath;
  }

  public void Start()
  {
    ...
    // Create the outbound filter queue if it doesn't exist.
    ...
    this.outQueue = QueueClient.CreateFromConnectionString(...);

    ...
    // Create the inbound and outbound queue clients.
    this.inQueue = QueueClient.CreateFromConnectionString(...);
  }

  public void OnPipeFilterMessageAsync(
    Func<BrokeredMessage, Task<BrokeredMessage>> asyncFilterTask, ...)
  {
    ...

    this.inQueue.OnMessageAsync(
      async (msg) =>
    {
      ...
      // Process the filter and send the output to the
      // next queue in the pipeline.
      var outMessage = await asyncFilterTask(msg);

      // Send the message from the filter processor
      // to the next queue in the pipeline.
      if (outQueue != null)
      {
        await outQueue.SendAsync(outMessage);
      }

      // Note: There's a chance that the same message could be sent twice
      // or that a message gets processed by an upstream or downstream
      // filter at the same time.
      // This would happen in a situation where processing of a message was
      // completed, it was sent to the next pipe/queue, and then failed
      // to complete when using the PeekLock method.
      // Idempotent message processing and concurrency should be considered
      // in a real-world implementation.
    },
    options);
  }

  public async Task Close(TimeSpan timespan)
  {
    // Pause the processing threads.
    this.pauseProcessingEvent.Reset();

    // There's no clean approach for waiting for the threads to complete
    // the processing. This example simply stops any new processing, waits
    // for the existing thread to complete, then closes the message pump
    // and finally returns.
    Thread.Sleep(timespan);

    this.inQueue.Close();
    ...
  }

  ...
}
```
The following code shows an Azure worker role named PipeFilterARoleEntry, defined in the PipeFilterA project in the sample solution.
`ServiceBusPipeFilter`类中的`Start`方法连接到一对输入和输出队列，`Close`方法断开与输入队列的连接。`OnPipeFilterMessageAsync`方法实际处理消息，该方法的`asyncFilterTask`参数指定要执行的处理。 `OnPipeFilterMessageAsync`方法等待输入队列中的传入消息，在每个消息到达时运行由`asyncFilterTask`参数指定代码，并将结果发布到输出队列。队列本身是由构造函数指定的。
案例解决方案使用一组角色实现过滤器。每个辅助角色都可以独立扩展，具体取决于其执行的业务处理的复杂程度或处理所需的资源。另外，可以并行运行每个辅助角色的多个实例以提高吞吐量。
以下代码显示了示例解决方案中`PipeFilterA`项目定义的名为`PipeFilterARoleEntry`的Azure辅助角色。

```c#
public class PipeFilterARoleEntry : RoleEntryPoint
{
  ...
  private ServiceBusPipeFilter pipeFilterA;

  public override bool OnStart()
  {
    ...
    this.pipeFilterA = new ServiceBusPipeFilter(
      ...,
      Constants.QueueAPath,
      Constants.QueueBPath);

    this.pipeFilterA.Start();
    ...
  }

  public override void Run()
  {
    this.pipeFilterA.OnPipeFilterMessageAsync(async (msg) =>
    {
      // Clone the message and update it.
      // Properties set by the broker (Deliver count, enqueue time, ...)
      // aren't cloned and must be copied over if required.
      var newMsg = msg.Clone();

      await Task.Delay(500); // DOING WORK

      Trace.TraceInformation("Filter A processed message:{0} at {1}",
        msg.MessageId, DateTime.UtcNow);

      newMsg.Properties.Add(Constants.FilterAMessageKey, "Complete");

      return newMsg;
    });

    ...
  }

  ...
}
```
该角色包含一个`ServiceBusPipeFilter`对象。角色中的`OnStart`方法连接到接收输入消息和发布输出消息的队列（队列的名称在`Constants`类中定义）。Run方法调用`OnPipeFilterMessagesAsync`方法对接收到的每条消息执行一些处理（在本例中，处理是通过等待一段时间来模拟的）。处理完成后，构造一个包含结果的新消息（在这种情况下，输入消息具有添加的定制属性），并将此消息发布到输出队列。
示例代码在`PipeFilterB`项目中包含另一个名为`PipeFilterBRoleEntry`的辅助角色。除了在`Run`方法中执行不同的处理外，该角色与`PipeFilterARoleEntry`相似。示例解决方案中将这两个角色组合起来构建一个管道，`PipeFilterARoleEntry`角色的输出队列是`PipeFilterBRoleEntry`角色的输入队列。
该示例解决方案还提供了两个名为`InitialSenderRoleEntry`（在`InitialSender`项目中）和`FinalReceiverRoleEntry`（在`FinalReceiver`项目中）的其它角色。`InitialSenderRoleEntry`角色在管道中提供初始消息。`OnStart`方法连接到单个队列，`Run`方法将一个方法发送到此队列。此队列是`PipeFilterARoleEntry`角色使用的输入队列，因此向其发送消息会导致消息被`PipeFilterARoleEntry`角色接收和处理。处理的消息之后通过`PipeFilterBRoleEntry`角色。
`FinalReceiveRoleEntry`角色的输入队列是`PipeFilterBRoleEntry`角色的输出队列。`FinalReceiveRoleEntry`角色中的Run方法（如下所示）接收消息并执行一些最终处理。然后，将管道中的过滤器添加的自定义属性的值写入跟踪输出。
```c#
public class FinalReceiverRoleEntry : RoleEntryPoint
{
  ...
  // Final queue/pipe in the pipeline to process data from.
  private ServiceBusPipeFilter queueFinal;

  public override bool OnStart()
  {
    ...
    // Set up the queue.
    this.queueFinal = new ServiceBusPipeFilter(...,Constants.QueueFinalPath);
    this.queueFinal.Start();
    ...
  }

  public override void Run()
  {
    this.queueFinal.OnPipeFilterMessageAsync(
      async (msg) =>
      {
        await Task.Delay(500); // DOING WORK

        // The pipeline message was received.
        Trace.TraceInformation(
          "Pipeline Message Complete - FilterA:{0} FilterB:{1}",
          msg.Properties[Constants.FilterAMessageKey],
          msg.Properties[Constants.FilterBMessageKey]);

        return null;
      });
    ...
  }

  ...
}
```

## 相关模式和指南
以下模式和指南在实现此模式时很有用：
* 此模式的示例在[GitHub上](https://github.com/mspnp/cloud-design-patterns/tree/master/pipes-and-filters)可以找到。
* [竞争消费者模式](competing-consumers.html)。一个管道可以包含一个或多个过滤器的多个实例。这种方法对于慢速过滤器的并行运行实例非常有用，使系统能够分散负载并提高吞吐量。过滤器的每个实例将与其它实例竞争输入，过滤器的两个实例不应该处理相同的数据。链接中的模式提供这种方法的解释。
* [计算资源合并模式](compute-resource-consolidation.html)。将可以一起放大到相同的过程中的过滤器分组也是可能的。链接提供了更多有关此策略的优点和折衷的信息。
* [补偿交易模式](https://docs.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction)。过滤器可以实现为可以反转的操作，或者具有补偿操作，以在发生故障的情况下将状态恢复到先前的版本。链接中的模式解释如何实现这一点，以保持或实现最终的一致性。
* Jonathan Oliver的博客上介绍的[幂等模式](http://blog.jonathanoliver.com/idempotency-patterns/)。