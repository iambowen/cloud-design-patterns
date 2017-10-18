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
读写存储器的分离也允许每个存储器适当地伸缩以匹配负载。例如，读存储器通常遇到比写存储器高得多的负载。
当查询/读模型包含非规范化数据（请参阅* [物化视图模式](materialized-view.md)）时，为应用程序中的每个视图读取数据或查询系统中的数据时，性能将最大化。

### 问题和注意事项

在决定如何实现此模式时，请考虑以下几点：
* 将数据存储分为单独的物理存储以进行读写操作可以提高系统的性能和安全性，但它增加弹性和最终一致性的复杂性。必须更新读模型存储以反映对写模型存储的更改，并且难以检测用户何时发出请求基于陈旧数据的读取请求，这样的操作无法完成。
> 有关最终一致性的描述，请参阅[数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。
* 考虑将CQRS应用于系统最有价值的部分。
* 部署最终一致性的典型方法是将事件溯源与CQRS结合使用，写模型是仅由执行命令驱动的附加事件流。这些事件用于更新充当读模型的物化视图。更多相关信息，请参阅[事件溯源和CQRS](https://msdn.microsoft.com/library/dn568103.aspx#EventSourcingandCQRS)。

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

CQRS模式经常与事件溯源模式一起使用。基于CQRS的系统使用单独的读写数据模型，每个模型针对相关任务进行定制，并且通常位于物理上分开的存储中。与事件溯源模式一起使用时，事件存储是写模型，并且是合法的信息来源。基于CQRS的系统的读模型提供了数据的物化视图，通常是高度非规范化的视图。这些视图是针对应用程序的接口和显示要求量身定制的，有助于最大化显示和查询性能。
使用事件流作为写存储，而不是某个时间点的实际数据，避免了单个聚合的更新冲突，并最大限度地提高了性能和可扩展性。事件可以以异步的方式生成用于填充读存储数据的物化视图。
由于事件存储是合法的信息来源，因此可以删除物化视图并重播所有过去的事件，以便在系统演进或者读模型必须更改时，创建当前状态的新表示。物化视图实际上是数据持久的只读缓存。

使用CQRS结合事件溯源模式时，请考虑以下几点：

* 与读写存储分离的任何系统一样，基于此模式的系统都是最终一致的。正在生成的事件和正在更新的数据存储之间会有一些延迟。
* 该模式增加了复杂性，因为必须创建代码来启动和处理事件，并组合或更新查询或读模型所需的适当视图或对象。当采用事件采购模式时，CQRS模式的复杂性导致实现的困难性，同时需要采用不同的方法来设计系统。但是，事件溯源更容易对域进行建模，因为保留了数据更改的意图，更容易地重建视图或创建新的视图，。
* 通过重放和处理特定实体或实体集合的事件，生成用于读模型或数据预测中的物化视图可能需要大量的时间和资源去处理。尤其是需要长时间的总结或分析值情况下，因为所有相关事件可能都需要检查。解决这个问题需要通过按计划的间隔实现数据的快照，例如已经发生的特定操作的数量的总数或实体的当前状态。

## 案例

下面的例子是从使用不同定义的读和写模型的CQRS实现中提取出的部分代码。模型接口不规定底层数据存储的任何特性，并且它们可以独立进行演进和调优，因为这些接口是分开的。
以下代码显示了读模型定义。

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
该系统允许用户评价产品。应用程序代码使用代码中的`RateProduct`命令执行此操作。

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

系统使用`ProductsCommandHandler`类来处理应用程序发送的命令。客户端通常通过诸如队列的消息系统向域发送命令。命令处理程序接受这些命令并调用域接口的方法。每个命令的粒度旨在减少冲突请求的机会。以下代码显示了`ProductsCommandHandler`类的大概内容。

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

以下代码展示了写模型中的`IProductsDomain`接口。
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

还要注意`IProductsDomain`接口是如何包含域中含义的方法。通常，在CRUD环境中，这些方法具有通用名称，例如Save或者Update，并将DTO作为唯一的参数。CQRS方法可以设计为满足本组织业务和目录管理系统的需求。

## 相关模式和指南

以下模式和指南在实现此模式时很有用：
* 对于CQR​​S与其它架构风格的比较，请参见[架构风格]()和[CQRS架构风格](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/cqrs)。
* [数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)介绍了在使用CQRS模式时读写数据存储之间的最终一致性，以及这些问题如何解决，常见问题等。
* [数据分区指南](https://msdn.microsoft.com/library/dn589795.aspx)。介绍如何将CQRS模式中使用的读写数据存储区划分为可以单独管理和访问的分区，以提高可扩展性，减少争用并优化性能。
* [事件溯源模式](event-sourcing.md)。更详细地介绍了如何使用CQRS模式来实现事件溯源，以简化复杂域中的任务，同时提高性能，可扩展性和响应能力。以及如何为事务提供数据一致性，同时保持完整的审计跟踪和历史，实现补偿措施。
* [物化视图模式](materialized-view.md)。CQRS实现的读模型可以包含写模型数据的物化视图，或者可以使用读模型来生成物化视图。
* 模式与实践指南的[CQRS之旅](http://aka.ms/cqrs)。特别介绍了[命令查询责任分离模式](https://msdn.microsoft.com/library/jj591573.aspx)，以及这种模式何时有用，[结语：经验教训](https://msdn.microsoft.com/library/jj591568.aspx)部分可帮助你了解使用此模式时出现的一些问题。
* [Martin Fowler关于CQRS的博客](http://martinfowler.com/bliki/CQRS.html)，解释了模式的基础知识和与其它有用资源的链接。
* [Greg Young的博客](http://codebetter.com/gregyoung/)，探讨了CQRS模式。