# 调度者、代理、管理者模式

将一组分布式操作协调为单一的操作。如果有任何操作失败，请尝试透明地处理故障，否则撤销已执行的工作，从而使整个操作成功或失败。这可以为分布式系统增加弹性，使其能够恢复并重试由于瞬时异常，长时间故障和进程故障而失败的操作。

## 背景和问题

应用程序执行包含多个步骤的任务，其中一些步骤可能会调用远程服务或访问远程资源。 各个步骤可能是彼此独立的，但它们是由执行任务的应用程序逻辑编排的。
应用程序应确保任务尽可能运行完成，并解决访问远程服务或资源时可能发生的任何故障。故障发生的原因很多。例如，网络可能断开，通信可能中断，远程服务无响应或处于不稳定状态，或者远程资源暂时无法访问（可能是由于资源限制）。很多情况下故障将是暂时的，可以通过使用[重试模式](retry.html)来处理。
如果应用程序检测到更永久性的故障，不能轻易恢复，必须能够将系统恢复到一致状态，并确保整个操作的完整性。

## 解决方案

调度者-代理-管理者模式定义了以下角色。这些角色编排作为整体任务一部分执行的步骤。

调度者安排构成要执行的任务的步骤并编排其操作。这些步骤可以合并到一个流水线或工作流程中。调度者负责确保以正确的顺序执行此工作流程中的步骤。在执行每个步骤时，计划程序会记录工作流的状态，如“步骤尚未开始”，“步骤运行”或“步骤完成”。状态信息还应该包括该步骤完成所允许的时间的上限，称为完成时间。如果某个步骤需要访问远程服务或资源，则调度程序会调用相应的代理程序，并向其传递要执行的工作的详细信息。调度程序通常使用异步请求/响应消息与代理进行通信。这可以使用队列来实现，但是也可以使用其它分布式消息技术。

>调度者在[进程管理器模式](http://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html)中执行与进程管理器类似的功能。实际的工作流程通常由调度者控制的工作流引擎定义和实现。这种方法将工作流中的业务逻辑从调度者中分离出来。

* 代理包含封装调用远程服务的逻辑，或者访问由任务中的步骤引用的远程资源。每个代理通常将调用包装为单个服务或资源，实现相应的错误处理和重试逻辑（受到超时限制的限制，稍后会介绍）。如果调度程序运行的工作流中的步骤跨越不同的步骤使用多个服务和资源，则每个步骤都可能引用不同的代理（这是模式的实现细节）。

* 管理者监视调度者执行的任务中的步骤状态。定期运行（系统指定的频率），并检查调度者维护的步骤的状态。如果检测到任何超时或失败，则会安排相应的代理恢复该步骤或执行适当的补救措施（这可能涉及修改步骤的状态）。请注意，恢复或补救措施由调度者和代理执行。管理者应该简单地要求执行这些动作。

调度者、代理和管理者是逻辑组件，其物理实现取决于所使用的技术。例如，几个逻辑代理可能作为为单个Web服务的一部分实现。

调度者维护有关任务进度以及持久化数据存储中每个步骤的状态（称为状态存储）的信息。管理者可以使用这些信息来帮助确定步骤是否失败。下图说明了调度者，代理，管理者和状态存储之间的关系。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/scheduler-agent-supervisor-pattern.png)

>该图是模式的简化版本。在一个真正的实现中，可能有许多的实例同时运行，每个任务都是一个子集。同样，系统可以运行每个代理的多个实例，甚至可以运行多个管理者。在这种情况下，管理者必须认真协调它们的工作，以确保它们不会为了恢复相同的失败步骤和任务而相互竞争。[领导者选举模式](leader-election.html)为这个问题提供了一个可能的解决方案。

当应用程序准备好运行任务时，它向调度者提交一个请求。调度者在状态存储区中记录有关任务及其步骤的初始状态信息（例如，步骤尚未开始），然后开始执行由工作流定义的操作。当调度者启动每一步时，它会更新有关状态存储中该步骤状态的信息（例如，步骤在运行中）。

如果某个步骤引用了远程服务或资源，则调度者将向相应的代理发送消息。该消息包含代理程序需要传递给服务或访问资源的信息，以及操作的完成时间。如果代理程序成功完成其操作，它将向计划程序返回一个响应。然后调度程序可以更新状态存储中的状态信息（例如，完成步骤）并执行下一步。这个过程一直持续到整个任务完成。

代理可以实现任何执行其工作所必需的重试逻辑。但是，如果代理在时间到期之前没有完成工作，则调度者将认为操作失败。在这种情况下，代理应该停止工作，不要试图将任何东西返回给调度者（甚至不会出现错误消息），或尝试任何形式的恢复。这样限制的原因是，在一个步骤超时或失败之后，可能会安排另一个代理实例运行失败步骤（稍后介绍此过程）。

如果代理失败，调度者将不会收到响应。这种模式并没有区分已经超时的步骤和真正失败的步骤。

如果某个步骤超时或失败，状态存储将包含一个记录，指示该步骤正在运行，但是已经过去了一段时间。管理者寻找这样的步骤，并试图恢复它们。一种可能的策略是管理者更新完整值以延长完成该步骤的可用时间，然后向调度者发送消息，以识别已超时的步骤。调度者然后可以尝试重复此步骤。但是，这种设计要求任务是幂等的。

如果监督者连续失败或超时，可能需要防止同一步骤重试。要做到这一点，管理者可以在状态存储中保存每个步骤的重试次数以及状态信息。如果计数超过预定的阈值，则管理者可以采取等待较长时间的策略，然后通知调度者应该重试此步骤，以期在此期间解决故障。或者，管理者可以通过执行[补偿交易模式](compensating-transaction.html)向调度者发送消息以请求撤消整个任务。这种方法将取决于调度者和代理提供实施成功完成的每个步骤的补偿操作的必要信息。

>管理者的目的不是监控调度者和代理，而是在失败时重新启动它们。系统的这个方面应该由运行这些组件的基础设施来处理。同样，管理者不应该知道调度者正在执行的任务的实际业务操作的内容（包括如果这些任务失败）。这是调度者实现的工作流逻辑的目的。管理者的唯一责任是确定一个步骤是否失败，并安排是重复执行还是将包含失败步骤的整个任务撤消。

如果调度者在发生故障后重新启动，或者由调度程序执行的工作流程意外终止，则调度程序应该能够确定在发生故障时正在处理的任何中转任务的状态，并准备从该调度程序中恢复此任务点。这个过程的实现细节可能是系统特定的。如果任务无法恢复，则可能需要撤消该任务已执行的工作。也可能需要实施[补偿性交易](compensating-transaction.html)。

这种模式的关键优势在于，系统在发生意外的暂时或不可恢复的故障时具有弹性。该系统可以构建为能自我修复。例如，如果代理或调度程序出现故障，则可以启动新代理，并且管理者可以安排要恢复的任务。如果管理者失败，另一个实例可以启动，并可以从发生故障的地方接管。如果管理者定期运行，则可以在预定义的时间间隔后自动启动新的实例。可以复制状态存储以达到更大程度的弹性。

## 问题和注意事项
在决定如何实现这种模式时，应该考虑以下几点：

* 这种模式可能很难实现，需要对系统的每种可能的故障模式进行彻底的测试。
* 调度者执行的恢复/重试逻辑非常复杂，取决于状态存储中保存的状态信息。在持久的数据存储中记录实施补偿性交易所需的信息也许是必要的。
* 管理者多久运行一次很重要。它应该经常运行，以防止任何失败的步骤长时间阻塞应用程序，但它不应该过多运行，以至于带来过多开销。
* 代理执行的步骤可以运行多次。实现这些步骤的逻辑应该是幂等的。

## 何时使用此模式

当在分布式环境（如云）中运行的进程必须对通信故障和/或操作故障具有弹性时，请使用此模式。
此模式可能不适于不调用远程服务或访问远程资源的任务。

## 案例

电子商务系统的Web应用程序已部署在Microsoft Azure上。用户可以运行此应用程序来浏览可用产品并下订单。用户界面作为Web角色运行，应用程序的订单处理元素被实现为一组辅助角色。订单处理逻辑的一部分涉及访问远程服务，系统的这一方面可能容易出现瞬态或更持久的故障。为此，设计人员使用调度者-代理-管理者模式来实现系统的订单处理元素。

当客户下订单时，应用程序会构造一条描述订单的消息，并将此消息发布到队列中。以工作角色运行的单独提交流程检索消息，将订单详细信息插入订单数据库，并在状态存储中为订单处理创建记录。请注意，插入订单数据库和状态存储是作为相同操作的一部分执行的。提交过程旨在确保两个插入一起完成。

提交过程为订单创建的状态信息包括：

* **订单ID**。订单数据库中订单的ID。

* **LockedBy**。处理订单的辅助角色的实例标识。运行调度程序的工作角色可能有多个当前实例，但每个订单只能由一个实例处理。

* **CompleteBy**。订单处理的时间。

* **ProcessState**。处理订单的任务的当前状态。可能的状态包括：
        * **待定**。订单已创建，但尚未开始处理。
        * **处理**。订单正在处理中。
        * **处理**。订单已成功处理。
        * **错误**。订单处理失败。

* **FailureCount**。已经尝试处理订单的次数。

在此状态信息中，`OrderID`字段从新订单的订单ID复制而来。 `LockedBy`和`CompleteBy`字段被设置为null，`ProcessState`字段被设置为Pending，并且`FailureCount`字段被设置为0。

>在这个例子中，订单处理逻辑相对简单，只有一个调用远程服务的步骤。在更复杂的多步骤场景中，提交过程可能涉及多个步骤，因此状态存储中将创建多个记录，每个记录描述单个步骤的状态。

调度者也作为辅助角色的一部分运行，并实现处理订单的业务逻辑。调度者轮询新订单的实例将检查`LockedBy`字段为空且`ProcessState`字段处于挂起状态的记录的状态存储。当调度者发现一个新的订单时，立即用自己的实例ID填充`LockedBy`字段，将`CompleteBy`字段设置为适当的时间，并将`ProcessState`字段设置为处理中。该代码被设计为独占和原子，以确保调度程序的两个并发实例不能尝试同时处理相同的订单。

然后，调度程序运行业务工作流以异步处理订单，并将状态存储中的`OrderID`字段中的值传递给它。处理订单的工作流程将从订单数据库中检索订单的详细信息并执行其工作。当订单处理工作流中的某个步骤需要调用远程服务时，它使用一个代理。工作流步骤使用一对充当请求/响应通道的Azure服务总线消息队列与代理进行通信。下图显示了该解决方案的概况。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/scheduler-agent-supervisor-solution.png)

从工作流步骤发送到代理的消息描述了订单并包含完成时间。如果代理在完成时间到期之前收到来自远程服务的响应，则会在工作流正在侦听的服务总线队列上发布回复消息。当工作流步骤接收到有效的回复消息时，它完成其处理，并且调度者设置要处理的订单状态的`ProcessState`字段。此时，订单处理已成功完成。

如果在代理收到远程服务响应之前完成时间到期，代理将简单地停止并终止处理订单。同样，如果处理订单的工作流程超过完成时间，它也会终止。在这两种情况下，状态存储中的订单状态都保持设置为处理状态，但是完成时间表示处理订单的时间已经过去，并且过程被视为失败。请注意，如果访问远程服务的代理或处理订单（或两者）的工作流程意外终止，则状态存储中的信息将再次保持设置为处理状态，并最终将具有到期的完成时间值。

如果代理在尝试联系远程服务时检测到不可恢复的，非暂时性故障，它可以将错误响应发送回工作流。调度者可以将订单的状态设置为错误，并引发提醒操作员的事件。然后操作员可以尝试手动解决失败的原因，然后重新提交失败的处理步骤。

管理者定期检查状态存储，查找完成时间过期的订单。如果管理者发现一条记录，则会增加`FailureCount`字段。如果`FailureCount`低于指定的阈值，管理者将`LockedBy`字段重置为空，用新的到期时间更新`CompleteBy`字段，并将`ProcessState`字段设置为未决定。调度者的一个实例可以拿起这个订单并像以前一样执行处理。如果`FailureCount`值超过指定的阈值，则认为失败的原因是非暂时性的。管理者将订单的状态设置为错误，并引发提醒操作员的事件。

>在这个例子中，管理者是在一个单独的辅助角色中实现的。可以使用各种策略来安排管理者任务运行，包括使用Azure Scheduler服务（不要与此模式中的调度者组件混淆）。关于Azure调度者服务的更多信息，请访问[调度者](https://azure.microsoft.com/services/scheduler/)页面。

尽管在本例中没有显示，但是调度者可能需要让提交订单的应用程序知道订单的进度和状态。应用程序和调度程序是相互隔离的，以消除它们之间的依赖关系。应用程序不知道调度者的哪个实例正在处理订单，并且调度者不知道哪个特定的应用程序实例公布了订单。

为了能够报告订单状态，应用程序可以使用自己的专用响应队列。这个响应队列的细节将作为发送到提交过程的请求的一部分，包括在状态存储中的信息。调度者然后将消息发送到此队列，指示订单的状态（请求接收，订单完成，订单失败等等）。它应该在这些消息中包含订单ID，以便关联应用程序的原始请求。

## 相关的模式和指南

以下模式和指南在实施这种模式时可能是相关的：

* [重试模式](retry.md)。代理可以使用此模式来透明地重试访问以前失败的远程服务或资源的操作。当期望失败的原因是暂时的并且可以纠正时使用。
* [断路器模式](circuit-breaker.md)。代理可以使用此模式处理连接到远程服务或资源时需要花费不同时间的错误。
* [补偿交易模式](compensating-transaction.md)。如果调度者执行的工作流程无法成功完成，则可能需要撤消之前执行的任何工作。补偿交易模式描述了如何实现遵循最终一致性模型的操作。这些类型的操作通常由执行复杂业务流程和工作流程的调度者来实现。
* [异步消息入门](https://msdn.microsoft.com/library/dn589781.aspx)。调度者-代理-管理者模式中的组件通常彼此解耦，并异步通信。本文介绍了一些可用于基于消息队列实现异步通信的方法。
* [领导者选举模式](leader-election.md)。可能需要协调管理者的多个实例的操作，以防止它们尝试恢复相同的失败进程。领导者选举模式介绍了如何做到这一点。
* Clemens Vasters博客上的[云体系结构：调度者-代理-管理者模式](https://blogs.msdn.microsoft.com/clemensv/2010/09/27/cloud-architecture-the-scheduler-agent-supervisor-pattern/)
* [进程管理器模式](http://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html)
* [参考6：Saga的传奇](https://msdn.microsoft.com/library/jj591569.aspx)。介绍CQRS模式如何使用流程管理器（CQRS之旅指南的一部分）的案例。
* [Microsoft Azure Scheduler](https://azure.microsoft.com/services/scheduler/)