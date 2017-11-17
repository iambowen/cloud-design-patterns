# 优先级队列模式

对发送给服务的请求进行优先级排序，以便更快地接收和处理具有更高优先级的请求。这种模式在为各个客户提供不同服务级别保证的应用程序中非常有用。

## 问题和背景

应用程序可以将特定任务委托给其它服务，例如执行后台处理或与其它应用程序或服务集成。在云中，消息队列通常用于将任务委托给后台处理。在许多情况下，订单请求被服务接收并不重要。 但在某些情况下，有必要优先考虑具体的要求。这些请求应该早于由应用程序先前发送的低优先级请求处理。

## 解决方案

队列通常是先入先出（FIFO）结构，而消费者通常按照发送到队列的相同顺序接收消息。但是，某些消息队列支持优先消息。发布消息的应用程序可以分配优先级，队列中的消息将自动重新排序，以便先收到优先级更高的消息。下图说明了具有优先消息的队列。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/priority-queue-pattern.png)

>大多数消息队列实现支持多个消费者（遵循[竞争消费者模式](competing-consumers.html)），并且消费者进程的数量可以根据需求而放大或缩小。


在不支持基于优先级的消息队列的系统中，另一种解决方案是为每个优先级维护一个单独的队列。应用程序负责将消息发布到适当的队列。每个队列可以有一个单独的消费者池。较高优先级的队列可以拥有比较低优先级的队列运行速度更快的硬件。下图说明了为每个优先级使用单独的消息队列。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/priority-queue-separate.png)

这种策略的一个变种就是拥有单个消费者池，先检查高优先级队列中的消息，然后才开始从低优先级队列中获取消息。在使用单个消费者进程池的解决方案（具有支持不同优先级消息的单个队列或者多个处理单个优先级消息的队列）之间存在一些语义差异，以及使用多个队列，每个队列都有一个单独的池。

在单池方法中，优先级较高的消息总是在较低优先级的消息之前接收和处理。从理论上讲，优先级非常低的消息可能会不断被取代，永远不会被处理。在多池方法中，总会处理较低优先级的消息，只是速度不如较高优先级的消息（取决于池的相对大小和可用资源）。

使用优先级排队机制有如下的好处：
* 允许应用程序满足业务要求，要求优先考虑可用性或性能，例如为特定的客户群提供不同级别的服务。
* 这可以帮助降低运维成本。单一队列方法可以根据需要缩减使用者的数量。高优先级的消息仍然会首先处理（尽管可能更慢），低优先级的消息可能会延迟更长的时间。如果已经为每个队列实施了具有不同使用者池的多个消息队列方法，则可以减少较低优先级队列的使用者池，甚至可以暂停处理一些超低优先级队列，方法是停止所有使用者监听这些队列上的消息。
* 多消息队列方法可以通过根据处理需求对消息进行分区来最大化应用程序性能和可伸缩性。例如，重要的任务可以优先处理，由接受者立即运行，而不重要的后台任务可以在不太繁忙的时间运行的接收者处理。

## 问题和注意事项

在决定如何实现该模式时，请考虑以下几点：

在解决方案的上下文中定义优先级。例如，高优先级可能意味着消息应该在十秒内处理。确定处理高优先级项目的要求，以及为满足这些标准而分配的其它资源。

确定是否所有高优先级的项目必须在低优先级项目之前处理。如果只有单个消费者池处理消息，则必须提供一种机制，以便在优先级较高的消息可用时抢占并暂停处理低优先级消息的任务。

在多队列方法中，使用单个消费者进程池（每个队列监听所有队列而不是专用消费者池）时，消费者必须应用一种算法，以确保其始终为优先处理高优先级队列消息。

监控高优先级队列和低优先级队列的处理速度，以确保这些队列中的消息按预期的速率处理。

如果需要保证处理低优先级的消息，那么有必要实现具有多个消费者池的多消息队列方法。或者，在支持消息优先级的队列中，可以动态增加排队消息在老化时的优先级。但是，这种方法取决于提供此功能的消息队列。

为每个消息使用单独的队列优先级对于具有少量定义明确的优先级的系统来说效果最佳。

消息优先级可以由系统逻辑确定。例如，它们可以被指定为“付费客户”或“非付费客户”，而不是具有明确的高优先级和低优先级消息。根据业务模型，系统可以分配更多资源处理来自付费客户的消息。

可能存在与检查消息队列相关联的金融和处理成本（一些商业消息传递系统每次发布或检索消息，并且每次查询消息的队列时收取少量费用）。此成本在检查多个队列时会增加。
可以根据正在服务的队列的长度动态调整消费者池的大小。更多信息请参阅[自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。

## 何时使用该模式

该模式适用于以下场景：

* 系统必须处理具有不同优先级的多个任务。
* 不同的用户或租户应该享有不同的优先权。

## 案例

Microsoft Azure不原生支持通过排序自动确定消息的优先级的排队机制。但是，它确实提供了支持提供消息过滤的排队机制的Azure服务总线主题和订阅，以及广泛的灵活功能，使其成为大多数优先级队列实现的理想选择。

Azure解决方案可以实现应用程序可以将消息发布到的服务总线主题，方式与队列相同。消息可以包含应用程序定义的自定义属性形式的元数据。服务总线订阅可以与该主题相关联，并且这些订阅可以基于消息的属性来过滤消息。当应用程序向主题发送消息时，消息被定向到适当的订阅，消费者可以读取消息。消费者进程可以使用与消息队列相同的语义从订阅中检索消息（订阅是逻辑队列）。下图说明了使用Azure Service Bus主题和订阅实现优先级队列。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/priority-queue-service-bus.png)

在上图中，应用程序创建了几个消息，并在每个消息中分配一个名为`Priority`的自定义属性，其值为`High`或`Low`。应用程序将这些消息发布到一个主题。该主题有两个关联的订阅，通过检查`Priority`属性过滤邮件。一个订阅接受`Priority`属性设置为`High`的消息，另一个接受`Priority`属性设置为`Low`的消息。消费者池从每个订阅中读取消息。高优先级的订阅具有更大的池，并且这些消费者可以运行在比低优先级池中的消费者更多的可用资源的更强大的计算机上。

请注意，在这个例子中，没有什么特别的地方可以指定高优先级和低优先级的消息。它们只是在每条消息中指定为属性的标签，用于将消息定向到特定的订阅。如果需要额外的优先级，创建更多的订阅和消费者进程池来处理这些优先级相对容易。

[GitHub上](https://github.com/mspnp/cloud-design-patterns/tree/master/priority-queue)提供的`PriorityQueue`解决方案包含了这种方法的实现。该解决方案包含两个名为`PriorityQueue.High`和`PriorityQueue.Low`的工作角色项目。这些角色继承自`PriorityWorkerRole`类，包含`OnStart`方法中用于连接到指定订阅的功能。

`PriorityQueue.High`和`PriorityQueue.Low` 工作者角色连接到由其配置定义的不同订阅。管理员可以配置不同数量的角色来运行。通常情况下，`PriorityQueue.High`工作者角色的实例将比`PriorityQueue.Low`工作者角色更多。

`PriorityWorkerRole`类中的`Run`方法安排在队列中接收的每个消息都运行虚拟`ProcessMessage`方法（也在`PriorityWorkerRole`类中定义）。以下代码显示了`Run`和`ProcessMessage`方法。`QueueManager`类在`PriorityQueue.Shared`项目中定义，它提供了使用Azure服务总线队列的辅助方法。
```c#
public class PriorityWorkerRole : RoleEntryPoint
{
  private QueueManager queueManager;
  ...

  public override void Run()
  {
    // Start listening for messages on the subscription.
    var subscriptionName = CloudConfigurationManager.GetSetting("SubscriptionName");
    this.queueManager.ReceiveMessages(subscriptionName, this.ProcessMessage);
    ...;
  }
  ...

  protected virtual async Task ProcessMessage(BrokeredMessage message)
  {
    // Simulating processing.
    await Task.Delay(TimeSpan.FromSeconds(2));
  }
}
```
`PriorityQueue.High`和`PriorityQueue.Low`工作角色都覆盖了`ProcessMessage`方法的默认功能。下面的代码显示了`PriorityQueue.High`worker角色的`ProcessMessage`方法。
```C#
protected override async Task ProcessMessage(BrokeredMessage message)
{
  // Simulate message processing for High priority messages.
  await base.ProcessMessage(message);
  Trace.TraceInformation("High priority message processed by " +
    RoleEnvironment.CurrentRoleInstance.Id + " MessageId: " + message.MessageId);
}
```
当应用程序将消息发送到与`PriorityQueue.High`和`PriorityQueue.Low`工作程序角色所使用的预订相关联的主题时，它会使用`Priority`自定义属性指定优先级，如下面的代码所示。代码（在`PriorityQueue.Sender`项目的`WorkerRole`类中实现）使用`QueueManager`类的`SendBatchAsync`帮助器方法批量发送消息到主题。
```C#
// Send a low priority batch.
var lowMessages = new List<BrokeredMessage>();

for (int i = 0; i < 10; i++)
{
  var message = new BrokeredMessage() { MessageId = Guid.NewGuid().ToString() };
  message.Properties["Priority"] = Priority.Low;
  lowMessages.Add(message);
}

this.queueManager.SendBatchAsync(lowMessages).Wait();
...

// Send a high priority batch.
var highMessages = new List<BrokeredMessage>();

for (int i = 0; i < 10; i++)
{
  var message = new BrokeredMessage() { MessageId = Guid.NewGuid().ToString() };
  message.Properties["Priority"] = Priority.High;
  highMessages.Add(message);
}

this.queueManager.SendBatchAsync(highMessages).Wait();
```

## 相关模式和指南

以下模式和指南在实现此模式时很有用：
* 此模式的案例代码可以在[GitHub](https://github.com/mspnp/cloud-design-patterns/tree/master/priority-queue)上下载。
* [异步消息入门](https://msdn.microsoft.com/library/dn589781.aspx)。处理请求的消费者服务可能需要向发布请求的应用程序实例发送回复。该指南提供了有关可用于实施请求/响应消息传递的策略的信息。
* [竞争消费者模式](consumer-competing.md)。为了增加队列的吞吐量，可以让多个使用者在同一队列上侦听，并且并行处理任务。这些消费者将争夺消息，但只有一个消费者能够处理每个消息。该指南提供了更多关于实施这种方法的好处和权衡的信息。
* [限流模式](throttling.md)。可以通过使用队列来实现限流。高优先级消息可用于确保来自关键应用程序或高价值客户运行的应用程序的请求优先运行。
* [自动缩放指南](https://msdn.microsoft.com/library/dn589774.aspx)。根据队列的长度，可以扩展处理队列的消费者进程池的大小。此策略可以帮助提高性能，特别是处理高优先级消息的池。
* Abhishek Lal的博客上关于[服务总线的企业集成模式](http://abhishekrlal.com/2013/01/11/enterprise-integration-patterns-with-service-bus-part-2/)
