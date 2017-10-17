# 命令和查询责任分离模式

使用不同的接口，分离更新数据的操作和读取数据的操作。可以最大限度地提高性能，可扩展性和安全性。通过更高的灵活性支持系统随时间演进，并防止更新命令在域级别引起合并冲突。

## 背景和问题

在传统的数据管理系统中，命令（对数据的更新）和查询（对数据的请求）是针对单个数据存储库中的同一组实体执行的。这些实体可以是关系数据库（如SQL Server）中的一个或多个表中的行的子集。
通常在这些系统中，所有创建，读取，更新和删除（CRUD）操作都应用于实体的相同表示。例如，通过数据访问层（DAL）从数据存储器检索表示客户的数据传输对象（DTO）并在屏幕上显示。用户更新DTO的某些字段（可能通过数据绑定），然后DAL保存DTO到数据存储区。 同样的DTO用于读写操作。下图介绍了传统的CRUD架构。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/command-and-query-responsibility-segregation-cqrs-tradition-crud.png)

当只有有限的业务逻辑被应用于数据操作时，传统的CRUD设计才能正常工作。开发工具提供的脚手架可以非常快速地创建数据访问代码，然后可以根据需要定制。

然而，传统的CRUD方法有一些缺点：
* 这通常意味着数据的读取和写入表示之间存在不匹配的情况，例如必须正确更新的附加列或属性，即使操作不要求这部分。
* 当记录锁定在协作域中的数据存储中时，数据争用就会发生风险，其中多个角色在同一组数据上并行操作。或者在并发更新使用乐观锁时引起的冲突。这些风险随着系统的复杂性和吞吐量的增加而增加。此外，由于数据存储和数据访问层的负载以及检索信息所需的查询的复杂性，传统的方法可能会对性能产生负面影响。
* 让管理安全性和权限更加复杂，因为每个实体都受读写操作的限制，这可能会将数据暴露在错误的上下文中。
> 为了更深入地了解CRUD方法的限制，请参阅[只有可以承担时使用CRUD](https://blogs.msdn.microsoft.com/maarten_mullender/2004/07/23/crud-only-when-you-can-afford-it-revisited/)

## 解决方案

命令和查询责任分离（CQRS）是一种模式，它通过使用单独的接口来隔离更新数据（命令）的操和读取数据（查询）的操作。这意味着用于查询和更新的数据模型是不同的。然后，模型可以如下图所示被隔离，尽管这不是绝对要求。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/command-and-query-responsibility-segregation-cqrs-basic.png)

与基于CRUD的系统中使用的单一数据模型相比，CQRS的系统中使用单独的查询和更新模型来简化设计和实现。然而，与CRUD设计不同，CQRS代码的缺点是不能使用脚手架自动生成。

用于读取数据的查询模型和用于写入数据的更新模型可以访问相同的物理存储，也许通过使用SQL视图或通过快速生成映射。然而，通常我们会将数据分成不同的物理存储，以最大限度地提高性能，可扩展性和安全性，如下图所示。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/command-and-query-responsibility-segregation-cqrs-separate-stores.png)

读存储可以是写存储的只读副本，或者读写存储可以具有完全不同的结构。使用读取存储的多个只读副本可以大大提高查询性能和应用程序UI响应，特别是在只读副本靠近应用程序实例分布式场景中。某些数据库系统（SQL Server）提供了诸如故障转移副本的附加功能，以最大限度地提高可用性。
读写存储器的分离也允许每个存储器适当地伸缩以匹配负载。例如，读存储器通常遇到比写入存储器高得多的负载。
当查询/读取模型包含非规范化数据（请参阅* [物化视图模式](materialized-view.md)）时，为应用程序中的每个视图读取数据或查询系统中的数据时，性能将最大化。

### 问题和注意事项

Consider applying CQRS to limited sections of your system where it will be most valuable.
A typical approach to deploying eventual consistency is to use event sourcing in conjunction with CQRS so that the write model is an append-only stream of events driven by execution of commands. These events are used to update materialized views that act as the read model. For more information see Event Sourcing and CQRS.
在决定如何实现此模式时，请考虑以下几点：
* 将数据存储分为单独的物理存储以进行读写操作可以提高系统的性能和安全性，但它增加弹性和最终一致性的复杂性。必须更新读模型存储以反映对写模型存储的更改，并且难以检测用户何时发出请求基于陈旧数据的读取请求，这样的操作无法完成。
> 有关最终一致性的描述，请参阅[数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。
* 考虑将CQRS应用于系统最有价值的部分。
* 部署最终一致性的典型方法是将事件溯源与CQRS结合使用，写入模型是仅由执行命令驱动的附加事件流。这些事件用于更新充当读取模型的物化视图。更多相关信息，请参阅[事件溯源和CQRS](https://msdn.microsoft.com/library/dn568103.aspx#EventSourcingandCQRS)。

## 何时使用该模式

在以下场景使用此模式：

* 在相同数据上并行执行多个操作的协作领域。 CQRS允许以适当的粒度定义命令，以最小化领域级别的合并冲突（即使出现任何冲突也可以由命令合并），即使看起来更新的是相同类型的数据。
* 基于任务的用户界面，其中用户通过一系列步骤或复杂领域模型引导复杂的进程。此外，对于已经熟悉域驱动设计（DDD）技术的团队来说，这是有用的。写模型具有完整的命令处理栈，业务逻辑，输入验证和业务验证，以确保写模型中的每个聚合（每个相关对象的集群作为数据更改的单位处理）中的所有内容始终保持一致。读模型没有业务逻辑或验证堆栈，并且仅返回用于视图模型的DTO。读模型最终与写模型一致。
* 数据读取性能必须与数据写入的性能分开进行优化，特别是当读/写比率非常高，以及需要水平缩放时。例如，在许多系统中，读操作的次数是写操作的许多倍。为了适应这一点，请考虑扩展读模型，仅在一个或几个实例上运行写模型。少量的写入模型实例也有助于最小化合并冲突的发生。
* 一个开发团队可以专注于作为写模型一部分的复杂域模型的场景，另一个团队可以专注于读模型和用户界面。
* 系统预期随着时间的推移而演进的场景，可能包含多个版本的模型，或业务规则定期更改的情况。
* 与其它系统集成，特别是与事件溯源相结合，其中某个子系统的临时故障不会影响其它系统的可用性。

在以下场景不推荐使用此模式：
* 领域或业务规则简单的地方。
* 简单的CRUD风格的用户接口和相关的数据访问操作就足够了。
* 用于整个系统的实现。CQRS对于整体数据管理方案的特定组件可能是有用的，对于不需要它的地方则增加相当的、不必要的复杂性。

## 事件溯源和CQRS

The CQRS pattern is often used along with the Event Sourcing pattern. CQRS-based systems use separate read and write data models, each tailored to relevant tasks and often located in physically separate stores. When used with the Event Sourcing pattern, the store of events is the write model, and is the official source of information. The read model of a CQRS-based system provides materialized views of the data, typically as highly denormalized views. These views are tailored to the interfaces and display requirements of the application, which helps to maximize both display and query performance.
Using the stream of events as the write store, rather than the actual data at a point in time, avoids update conflicts on a single aggregate and maximizes performance and scalability. The events can be used to asynchronously generate materialized views of the data that are used to populate the read store.
Because the event store is the official source of information, it is possible to delete the materialized views and replay all past events to create a new representation of the current state when the system evolves, or when the read model must change. The materialized views are in effect a durable read-only cache of the data.
When using CQRS combined with the Event Sourcing pattern, consider the following:
As with any system where the write and read stores are separate, systems based on this pattern are only eventually consistent. There will be some delay between the event being generated and the data store being updated.
The pattern adds complexity because code must be created to initiate and handle events, and assemble or update the appropriate views or objects required by queries or a read model. The complexity of the CQRS pattern when used with the Event Sourcing pattern can make a successful implementation more difficult, and requires a different approach to designing systems. However, event sourcing can make it easier to model the domain, and makes it easier to rebuild views or create new ones because the intent of the changes in the data is preserved.
Generating materialized views for use in the read model or projections of the data by replaying and handling the events for specific entities or collections of entities can require significant processing time and resource usage. This is especially true if it requires summation or analysis of values over long periods, because all the associated events might need to be examined. Resolve this by implementing snapshots of the data at scheduled intervals, such as a total count of the number of a specific action that have occurred, or the current state of an entity.

## 案例

The following code shows some extracts from an example of a CQRS implementation that uses different definitions for the read and the write models. The model interfaces don't dictate any features of the underlying data stores, and they can evolve and be fine-tuned independently because these interfaces are separated.
The following code shows the read model definition.

```java
// Query interface
namespace ReadModel
{
  public interface ProductsDao
  {
    ProductDisplay FindById(int productId);
    ICollection<ProductDisplay> FindByName(string name);
    ICollection<ProductInventory> FindOutOfStockProducts();
    ICollection<ProductDisplay> FindRelatedProducts(int productId);
  }

  public class ProductDisplay
  {
    public int Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal UnitPrice { get; set; }
    public bool IsOutOfStock { get; set; }
    public double UserRating { get; set; }
  }

  public class ProductInventory
  {
    public int Id { get; set; }
    public string Name { get; set; }
    public int CurrentStock { get; set; }
  }
}
```
The system allows users to rate products. The application code does this using the RateProduct command shown in the following code.
```java
public interface ICommand
{
  Guid Id { get; }
}

public class RateProduct : ICommand
{
  public RateProduct()
  {
    this.Id = Guid.NewGuid();
  }
  public Guid Id { get; set; }
  public int ProductId { get; set; }
  public int Rating { get; set; }
  public int UserId {get; set; }
}
```

The system uses the ProductsCommandHandler class to handle commands sent by the application. Clients typically send commands to the domain through a messaging system such as a queue. The command handler accepts these commands and invokes methods of the domain interface. The granularity of each command is designed to reduce the chance of conflicting requests. The following code shows an outline of the ProductsCommandHandler class.
```java
public class ProductsCommandHandler :
    ICommandHandler<AddNewProduct>,
    ICommandHandler<RateProduct>,
    ICommandHandler<AddToInventory>,
    ICommandHandler<ConfirmItemShipped>,
    ICommandHandler<UpdateStockFromInventoryRecount>
{
  private readonly IRepository<Product> repository;

  public ProductsCommandHandler (IRepository<Product> repository)
  {
    this.repository = repository;
  }

  void Handle (AddNewProduct command)
  {
    ...
  }

  void Handle (RateProduct command)
  {
    var product = repository.Find(command.ProductId);
    if (product != null)
    {
      product.RateProduct(command.UserId, command.Rating);
      repository.Save(product);
    }
  }

  void Handle (AddToInventory command)
  {
    ...
  }

  void Handle (ConfirmItemsShipped command)
  {
    ...
  }

  void Handle (UpdateStockFromInventoryRecount command)
  {
    ...
  }
}
```
The following code shows the IProductsDomain interface from the write model.

```java
public interface IProductsDomain
{
  void AddNewProduct(int id, string name, string description, decimal price);
  void RateProduct(int userId, int rating);
  void AddToInventory(int productId, int quantity);
  void ConfirmItemsShipped(int productId, int quantity);
  void UpdateStockFromInventoryRecount(int productId, int updatedQuantity);
}
```
Also notice how the IProductsDomain interface contains methods that have a meaning in the domain. Typically, in a CRUD environment these methods would have generic names such as Save or Update, and have a DTO as the only argument. The CQRS approach can be designed to meet the needs of this organization's business and inventory management systems.

## 相关模式和指南

The following patterns and guidance are useful when implementing this pattern:
For a comparison of CQRS with other architectural styles, see Architecture styles and CQRS architecture style.
Data Consistency Primer. Explains the issues that are typically encountered due to eventual consistency between the read and write data stores when using the CQRS pattern, and how these issues can be resolved.
Data Partitioning Guidance. Describes how the read and write data stores used in the CQRS pattern can be divided into partitions that can be managed and accessed separately to improve scalability, reduce contention, and optimize performance.
Event Sourcing Pattern. Describes in more detail how Event Sourcing can be used with the CQRS pattern to simplify tasks in complex domains while improving performance, scalability, and responsiveness. As well as how to provide consistency for transactional data while maintaining full audit trails and history that can enable compensating actions.
Materialized View Pattern. The read model of a CQRS implementation can contain materialized views of the write model data, or the read model can be used to generate materialized views.
The patterns & practices guide CQRS Journey. In particular, Introducing the Command Query Responsibility Segregation Pattern explores the pattern and when it's useful, and Epilogue: Lessons Learned helps you understand some of the issues that come up when using this pattern.
The post CQRS by Martin Fowler, which explains the basics of the pattern and links to other useful resources.
Greg Young’s posts, which explore many aspects of the CQRS pattern.