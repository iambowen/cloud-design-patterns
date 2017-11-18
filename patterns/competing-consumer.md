# 竞争消费者模式

启用多个并发消费者来处理在同一个消息通道上收到的消息。这使系统能够同时处理多个消息，以优化吞吐量，提高可扩展性和可用性，并平衡工作负载。

## 背景和问题

在云上运行的应用程序预计将处理大量的请求。常见的技术是让应用程序通过消息系统传递给以异步方式处理它们的另一个服务（消费者服务），而不是同步处理请求。这个策略有助于确保应用程序中的业务逻辑在处理请求时不被阻塞。

由于许多原因，请求的数量可能随着时间而显着变化。来自多个租户的用户活动的聚合请求突然增加，可能会导致无法预测的工作量。在高峰时间，系统可能需要每秒处理数百个请求，而在其它时间，数量可能非常小。另外，为处理这些请求所进行的工作的性质可能是高度可变的。使用消费者服务的单个实例会导致该实例充斥着请求，或者消息系统可能由于来自应用程序的大量消息而过载。为了处理这种波动的工作量，系统可以运行消费者服务的多个实例。但是，必须协调这些消费者，以确保每封邮件只能发送给单个消费者。工作负载还需要在消费者之间进行负载平衡，以防止实例成为瓶颈。

## 解决方案

使用消息队列来实现应用程序和客户服务实例之间的通信通道。应用程序以消息的形式向队列发送请求，消费者服务实例从队列接收消息并处理消息。这种方法使相同的消费者服务实例池能够处理来自任何应用程序实例的消息。下图说明了使用消息队列将工作分配给服务的实例。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/competing-consumers-diagram.png)
该解决方案具有以下优点：
* 提供了一个负载均衡的系统，可处理由应用程序实例发送的不同数量的请求。队列充当应用程序实例和使用者服务实例之间的缓冲区。这可以帮助最小化应用程序和服务实例的可用性和响应性的影响，如[基于队列的负载均衡模式](queue-based-load-leveling.html)所述。处理需要一些长时间运行才能处理的消息不会阻止消费者服务的其它实例同时处理其它消息。
* 提高了可靠性。如果一个生产者直接与消费者沟通，而不是使用这种模式，但不监督消费者，那么如果消费者失败，消息很可能会丢失或者不能被处理。在这种模式下，消息不会发送到特定的服务实例。失败的服务实例不会阻塞生产者，消息可以被任何工作服务实例处理。
* 不需要消费者之间或生产者与消费者实例之间的复杂协调。消息队列确保每个消息至少被传递一次。
* 可扩展。当消息量波动时，系统可以动态地增加或减少消费者服务的实例的数量。
* 如果消息队列提供事务性读取操作，则可以提高弹性。如果消费者服务实例作为事务操作的一部分来读取和处理消息，并且消费者服务实例失败，则该模式可以确保消息将被返回到队列以被消费者服务的另一实例拾取和处理。
## 问题和注意事项

在决定如何实现该模式时，请考虑以下几点：
* 消息排序。不能保证消费者服务实例接收消息的顺序，也不一定反映消息的创建顺序。设计系统以确保消息处理是幂等的，这将有助于消除对消息处理顺序的依赖。更多相关内容请参阅Jonathon Oliver博客上的[“幂等模式”](http://blog.jonathanoliver.com/idempotency-patterns/)。
>Microsoft Azure服务总线队列可以通过使用消息会话来保证先进先出的消息排序。更多相关信息请参阅[使用会话的消息传送模式](https://msdn.microsoft.com/magazine/jj863132.aspx)。
* 设计弹性服务。如果系统设计为检测并重新启动失败的服务实例，则可能有必要将服务实例执行的处理实现为幂等操作，以最大程度地减少单个消息被检索和处理多次的影响。
* 检测有毒消息。格式错误的消息或需要访问不可用资源的任务可能导致服务实例失败。系统应该阻止这些消息返回到队列中，并且将这些消息的细节捕获并存储在其它地方，以便在需要时可以对其进行分析。
* 处理结果。处理消息的服务实例与生成消息的应用程序逻辑完全分离，并且可能无法直接通信。如果服务实例生成的结果要传递回应用程序逻辑，则必须将此信息存储在两者均可访问的位置。为了防止应用程序逻辑检索不完整的数据，系统必须指示何时处理完成。
>如果使用Azure，则工作进程可以使用专用的消息应答队列将结果传递回应用程序逻辑。应用程序逻辑必须能够将这些结果与原始消息相关联。这个场景在[异步消息入门](https://msdn.microsoft.com/library/dn589781.aspx)中有更详细的介绍。
* 扩展消息传递系统。在一个大规模的解决方案中，单个消息队列可能被海量消息淹没，并成为系统中的一个瓶颈。在这种情况下，考虑将消息传递系统划分为将消息从特定的生产者发送到特定的队列，或使用负载均衡将消息分布到多个消息队列中。
* 确保消息传递系统的可靠性。需要可靠的消息传递系统来保证应用程序排队后不会丢失。这对于确保所有消息至少交付一次至关重要。

## 何时使用该模式
在以下情况使用此模式：

* 应用程序的工作负载分为可以异步运行的任务。
* 任务是独立的，可以并行运行。
* 工作量非常多变，需要一个可扩展的解决方案。
* 解决方案必须提供高可用性，并且在任务处理失败时具有弹性。

以下情况可能不适合该模式：
* 不容易将应用程序的工作量分解成离散任务，或者任务之间存在高度的依赖关系。
* 任务必须同步执行，应用程序逻辑必须等待任务完成才能继续。
* 任务必须按照特定的顺序进行。
>某些消息系统支持会话，使生产者能够将消息分组在一起，并确保它们都由同一个消费者处理。这种机制可以与优先消息（如果支持的话）一起使用，以实现消息顺序的形式，消息按顺序从生产者传递给单个消费者。

## 案例

Azure提供了存储队列和服务总线队列，可以充当实现这种模式的机制。应用程序逻辑可以将消息发布到队列中，并且作为一个或多个角色的任务实现的消费者，可以从这个队列中检索并处理消息。为了保持弹性，服务总线队列使消费者在从队列中检索消息时使用`PeekLock`模式。这种模式实际上并没有删除该消息，而只是将其从其它消费者隐藏。原消费者在完成处理后可以删除该消息。如果消费者失败，查看锁定将超时，消息将再次变为可见，从而允许另一个消费者检索它。

>有关使用Azure服务总线队列的详细信息，请参阅服[务总线队列，主题和预订](https://msdn.microsoft.com/library/windowsazure/hh367516.aspx)。有关使用Azure存储队列的信息，请参阅[使用.NET上手Azure队列存储](https://azure.microsoft.com/documentation/articles/storage-dotnet-how-to-use-queues/)。

下面的代码来自GitHub上的`CompetingConsumers`解决方案中的`QueueManager`类，显示了如何通过在Web或Worker角色的`Start`事件处理程序使用`QueueClient`实例来创建队列。

```c#
private string queueName = ...;
private string connectionString = ...;
...

public async Task Start()
{
  // Check if the queue already exists.
  var manager = NamespaceManager.CreateFromConnectionString(this.connectionString);
  if (!manager.QueueExists(this.queueName))
  {
    var queueDescription = new QueueDescription(this.queueName);

    // Set the maximum delivery count for messages in the queue. A message
    // is automatically dead-lettered after this number of deliveries. The
    // default value for dead letter count is 10.
    queueDescription.MaxDeliveryCount = 3;

    await manager.CreateQueueAsync(queueDescription);
  }
  ...

  // Create the queue client. By default the PeekLock method is used.
  this.client = QueueClient.CreateFromConnectionString(
    this.connectionString, this.queueName);
}
```
下面的代码段显示了应用程序如何创建一批消息并将其发送到队列。
```C#
public async Task SendMessagesAsync()
{
  // Simulate sending a batch of messages to the queue.
  var messages = new List<BrokeredMessage>();

  for (int i = 0; i < 10; i++)
  {
    var message = new BrokeredMessage() { MessageId = Guid.NewGuid().ToString() };
    messages.Add(message);
  }
  await this.client.SendBatchAsync(messages);
}
```
以下代码显示了消费者服务实例如何通过遵循事件驱动方法接收来自队列的消息。`ReceiveMessages`方法的`processMessageTask`参数是一个委托，它引用在收到消息时运行的代码。这段代码是异步运行的。
```C#

private ManualResetEvent pauseProcessingEvent;
...

public void ReceiveMessages(Func<BrokeredMessage, Task> processMessageTask)
{
  // Set up the options for the message pump.
  var options = new OnMessageOptions();

  // When AutoComplete is disabled it's necessary to manually
  // complete or abandon the messages and handle any errors.
  options.AutoComplete = false;
  options.MaxConcurrentCalls = 10;
  options.ExceptionReceived += this.OptionsOnExceptionReceived;

  // Use of the Service Bus OnMessage message pump.
  // The OnMessage method must be called once, otherwise an exception will occur.
  this.client.OnMessageAsync(
    async (msg) =>
    {
      // Will block the current thread if Stop is called.
      this.pauseProcessingEvent.WaitOne();

      // Execute processing task here.
      await processMessageTask(msg);
    },
    options);
}
...

private void OptionsOnExceptionReceived(object sender,
  ExceptionReceivedEventArgs exceptionReceivedEventArgs)
{
  ...
}
```
请注意，自动缩放功能（如Azure中提供的功能）可用于在队列长度波动时启动和停止角色实例。更多相关内容，请参阅[自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。而且，角色实例和工作进程之间不需要保持一对一的对应关系 - 单个角色实例可以实现多个工作进程。更多相关信息，请参阅[计算资源合并模式](compute-resource-consolidation.html)。

## 相关的模式和指南

以下模式和指南在实施这种模式时可能是相关的：
* [异步消息入门](https://msdn.microsoft.com/library/dn589781.aspx)。消息队列是一种异步通信机制。如果消费者服务需要向应用程序发送回复，则可能需要实现某种形式的回复消息。异步消息入门介绍了如何使用消息队列实现请求/回复消息的信息。
* [自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。队列应用程序发布消息的长度各不相同，可能启动和停止消费者服务的实例。自动缩放可以帮助在峰值处理期间保持吞吐量。
* [计算资源合并模式](compute-resource-consolidation.md)。存在将消费者服务的多个实例合并到单个进程中以降低成本和管理开销的可能性。计算资源合并模式介绍了遵循这种方法的好处和权衡。
* [基于队列的负载均衡模式](queue-based-load-leveling.md)。引入消息队列可以为系统增加弹性，使服务实例能够处理来自应用程序实例的大量不同的请求。消息队列充当缓冲区，负载均衡。基于队列的负载均衡模式更详细地描述了这种情况。
* 和该模式相关的一个[示例应用程序](https://github.com/mspnp/cloud-design-patterns/tree/master/competing-consumers)。