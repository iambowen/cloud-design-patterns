# 事件溯源模式

使用只可追加的存储来记录对数据所进行的所有操作，而不是存储领域数据的当前状态。该存储作为记录系统可用于实现领域对象。通过避免数据模型与业务领域之间的同步，该模式简化了复杂领域下的工作，同时改善了性能、可扩展性和响应能力。还能够提供事务数据一致性，并维护了全部的审计记录与历史，可用于修正操作。

## 背景与问题

大部分应用都需要和数据打交道，典型的方法是由应用维护数据的当前状态，在用户使用时对数据进行更新。例如，在传统的增删改查（CRUD）模型下，典型的数据处理过程是，从数据库中读取该数据，做一些修改后再用新值去更新数据的当前状态——通常会使用带锁的事务。

这种CRUD方法有一些局限性：

* CRUD系统在数据库上直接执行更新操作，由于需要开销，会降低系统的性能与响应能力，限制可扩展性。
* 在一个由多个用户并发操作的领域中，数据更新更有可能会起冲突，因为更新操作发生在同一条单独的数据项上。
* 除非在某个单独的日志中存在额外的审计机制来记录每个操作的细节，否则历史记录会丢失。

> 关于对CRUD方法局限性的更深入了解，见[CRUD, Only When You Can Afford It](https://msdn.microsoft.com/library/ms978509.aspx)。

## 解决方案

事件溯源模式对于由一系列事件驱动产生的数据定义了一套处理方法，每个事件被记录在一个只可追加的存储中。应用代码发送一系列事件，这些事件命令式地描述了数据上产生的每个操作，并持久化到事件数据库中。每个事件代表对数据的一系列变化（例如`AddedItemToOrder`）。

这些事件持久化在一个事件数据库中（可信的数据源），作为记录数据当前状态的系统。事件数据库通常会发布这些事件，以便通知到消费者，并由消费者进行所需的处理。例如，消费者可以在其它系统中利用这些事件来初始化一些操作任务，或者执行完成整个操作所需的其它相关动作。注意，产生事件的应用代码是与订阅事件的系统相解耦的。

事件数据库所发布事件的通常用途是在应用改变实体状态时维护实体的物化视图，以及与外部系统集成。例如，一个系统可以维护所有客户订单的物化视图，用于生成用户界面的一部分。当应用中新增订单，添加或删除订单中的商品项，以及添加配送信息时，可以对描述这些变化的事件进行处理并更新这个[物化视图](patterns/materialized-view.md)。

此外，任何时间点上应用都可以读取到事件历史，用来物化某个实体的当前状态，只需要重新播放与该实体相关的所有事件即可。这可以发生在需要处理某个实体对象的物化请求时，或者发生在某个计划任务中，将实体的状态以物化视图形式存储起来，以便支持展示层的需要。

下图显示了该模式的概览，包括使用事件流的一些方式，比如创建物化视图，通过事件与外部应用和系统进行集成，以及通过重新播放事件来建立特定实体的当前状态。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/event-sourcing-overview.png)

事件溯源模式可以提供如下这些优点：

事件是不可变的，可通过只可追加的方式进行存储。用户界面、工作流或者初始化事件的过程都可以继续进行，处理事件的任务可以在后台运行。因此，再加上由于在整个事务处理过程中不存在冲突，便可极大地改善应用的性能与可扩展性，特别是对于展示层或用户界面。

事件是描述已发生动作的一些简单对象，包含了事件在描述动作时所需的任何相关数据。事件并不直接更新数据库。它们只是被简单地记录下来以便在合适的时间进行处理。这可以简化实现和管理。

鉴于对象关系不匹配时会造成复杂数据库表的难于理解，事件对于领域专家通常是有特殊意义的。数据库表是用于表示系统当前状态的人为概念，而不是所有已发生的事件。

事件可以帮助避免引起冲突的并发更新，因为它避免了直接更新数据库中对象的需要。尽管如此，领域对象仍然要设计得能够防止造成不一致状态。

只可追加的事件存储提供了一种审计记录，可用于监控在数据库上发生的动作，以物化视图方式或者任何时刻重新播放事件的方式重新生成当前状态，有助于系统测试和调试。此外，如果需要通过修正事件来取消某些变更，可以通过对历史变更进行反向操作进行，而如果模型中只是简单存储了当前状态的话就不行了。事件列表还可用来分析应用性能和检测用户行为趋势，或者获得其他有用的业务信息。

事件存储产生事件，任务执行操作来响应这些事件。事件与任务的解耦提供了灵活性与可扩展性。任务知道事件类型与事件数据，却并不知道触发这些事件的操作。此外，每个事件可以被多任务处理。这提供了与其它服务和系统集成的方便方法，只需要监听由事件存储产生的新事件就可以了。尽管如此，事件溯源倾向于发生在非常低的层次上，有可能需要再生成一些特定的集成事件。

> 事件溯源经常与CQRS模式一起使用，执行数据管理任务来响应事件，以及从存储的事件中产生物化视图。


## 问题与注意事项


Consider the following points when deciding how to implement this pattern:
The system will only be eventually consistent when creating materialized views or generating projections of data by replaying events. There's some delay between an application adding events to the event store as the result of handling a request, the events being published, and consumers of the events handling them. During this period, new events that describe further changes to entities might have arrived at the event store.


Note

See the Data Consistency Primer for information about eventual consistency.


The event store is the permanent source of information, and so the event data should never be updated. The only way to update an entity to undo a change is to add a compensating event to the event store. If the format (rather than the data) of the persisted events needs to change, perhaps during a migration, it can be difficult to combine existing events in the store with the new version. It might be necessary to iterate through all the events making changes so they're compliant with the new format, or add new events that use the new format. Consider using a version stamp on each version of the event schema to maintain both the old and the new event formats.


Multi-threaded applications and multiple instances of applications might be storing events in the event store. The consistency of events in the event store is vital, as is the order of events that affect a specific entity (the order that changes occur to an entity affects its current state). Adding a timestamp to every event can help to avoid issues. Another common practice is to annotate each event resulting from a request with an incremental identifier. If two actions attempt to add events for the same entity at the same time, the event store can reject an event that matches an existing entity identifier and event identifier.
There's no standard approach, or existing mechanisms such as SQL queries, for reading the events to obtain information. The only data that can be extracted is a stream of events using an event identifier as the criteria. The event ID typically maps to individual entities. The current state of an entity can be determined only by replaying all of the events that relate to it against the original state of that entity.
The length of each event stream affects managing and updating the system. If the streams are large, consider creating snapshots at specific intervals such as a specified number of events. The current state of the entity can be obtained from the snapshot and by replaying any events that occurred after that point in time. For more information about creating snapshots of data, see Snapshot on Martin Fowler’s Enterprise Application Architecture website and Master-Subordinate Snapshot Replication.
Even though event sourcing minimizes the chance of conflicting updates to the data, the application must still be able to deal with inconsistencies that result from eventual consistency and the lack of transactions. For example, an event that indicates a reduction in stock inventory might arrive in the data store while an order for that item is being placed, resulting in a requirement to reconcile the two operations either by advising the customer or creating a back order.
Event publication might be “at least once,” and so consumers of the events must be idempotent. They must not reapply the update described in an event if the event is handled more than once. For example, if multiple instances of a consumer maintain an aggregate an entity's property, such as the total number of orders placed, only one must succeed in incrementing the aggregate when an order placed event occurs. While this isn't a key characteristic of event sourcing, it's the usual implementation decision.


When to use this pattern
Use this pattern in the following scenarios:
When you want to capture intent, purpose, or reason in the data. For example, changes to a customer entity can be captured as a series of specific event types such as Moved home, Closed account, or Deceased.
When it's vital to minimize or completely avoid the occurrence of conflicting updates to data.
When you want to record events that occur, and be able to replay them to restore the state of a system, roll back changes, or keep a history and audit log. For example, when a task involves multiple steps you might need to execute actions to revert updates and then replay some steps to bring the data back into a consistent state.
When using events is a natural feature of the operation of the application, and requires little additional development or implementation effort.
When you need to decouple the process of inputting or updating data from the tasks required to apply these actions. This might be to improve UI performance, or to distribute events to other listeners that take action when the events occur. For example, integrating a payroll system with an expense submission website so that events raised by the event store in response to data updates made in the website are consumed by both the website and the payroll system.
When you want flexibility to be able to change the format of materialized models and entity data if requirements change, or—when used in conjunction with CQRS—you need to adapt a read model or the views that expose the data.
When used in conjunction with CQRS, and eventual consistency is acceptable while a read model is updated, or the performance impact of rehydrating entities and data from an event stream is acceptable.
This pattern might not be useful in the following situations:
Small or simple domains, systems that have little or no business logic, or nondomain systems that naturally work well with traditional CRUD data management mechanisms.
Systems where consistency and real-time updates to the views of the data are required.
Systems where audit trails, history, and capabilities to roll back and replay actions are not required.
Systems where there's only a very low occurrence of conflicting updates to the underlying data. For example, systems that predominantly add data rather than updating it.



Example
A conference management system needs to track the number of completed bookings for a conference so that it can check whether there are seats still available when a potential attendee tries to make a booking. The system could store the total number of bookings for a conference in at least two ways:
The system could store the information about the total number of bookings as a separate entity in a database that holds booking information. As bookings are made or canceled, the system could increment or decrement this number as appropriate. This approach is simple in theory, but can cause scalability issues if a large number of attendees are attempting to book seats during a short period of time. For example, in the last day or so prior to the booking period closing.
The system could store information about bookings and cancellations as events held in an event store. It could then calculate the number of seats available by replaying these events. This approach can be more scalable due to the immutability of events. The system only needs to be able to read data from the event store, or append data to the event store. Event information about bookings and cancellations is never modified.
The following diagram illustrates how the seat reservation subsystem of the conference management system might be implemented using event sourcing.


![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/event-sourcing-bounded-context.png)

The sequence of actions for reserving two seats is as follows:
The user interface issues a command to reserve seats for two attendees. The command is handled by a separate command handler. A piece of logic that is decoupled from the user interface and is responsible for handling requests posted as commands.
An aggregate containing information about all reservations for the conference is constructed by querying the events that describe bookings and cancellations. This aggregate is called SeatAvailability, and is contained within a domain model that exposes methods for querying and modifying the data in the aggregate.
Some optimizations to consider are using snapshots (so that you don’t need to query and replay the full list of events to obtain the current state of the aggregate), and maintaining a cached copy of the aggregate in memory.
The command handler invokes a method exposed by the domain model to make the reservations.
The SeatAvailability aggregate records an event containing the number of seats that were reserved. The next time the aggregate applies events, all the reservations will be used to compute how many seats remain.
The system appends the new event to the list of events in the event store.
If a user cancels a seat, the system follows a similar process except the command handler issues a command that generates a seat cancellation event and appends it to the event store.
As well as providing more scope for scalability, using an event store also provides a complete history, or audit trail, of the bookings and cancellations for a conference. The events in the event store are the accurate record. There is no need to persist aggregates in any other way because the system can easily replay the events and restore the state to any point in time.
You can find more information about this example in Introducing Event Sourcing.
Related patterns and guidance
The following patterns and guidance might also be relevant when implementing this pattern:
Command and Query Responsibility Segregation (CQRS) Pattern. The write store that provides the permanent source of information for a CQRS implementation is often based on an implementation of the Event Sourcing pattern. Describes how to segregate the operations that read data in an application from the operations that update data by using separate interfaces.
Materialized View Pattern. The data store used in a system based on event sourcing is typically not well suited to efficient querying. Instead, a common approach is to generate prepopulated views of the data at regular intervals, or when the data changes. Shows how this can be done.
Compensating Transaction Pattern. The existing data in an event sourcing store is not updated, instead new entries are added that transition the state of entities to the new values. To reverse a change, compensating entries are used because it isn't possible to simply reverse the previous change. Describes how to undo the work that was performed by a previous operation.
Data Consistency Primer. When using event sourcing with a separate read store or materialized views, the read data won't be immediately consistent, instead it'll be only eventually consistent. Summarizes the issues surrounding maintaining consistency over distributed data.
Data Partitioning Guidance. Data is often partitioned when using event sourcing to improve scalability, reduce contention, and optimize performance. Describes how to divide data into discrete partitions, and the issues that can arise.
Greg Young’s post Why use Event Sourcing?.