## 缓存辅助模式

按需将数据从数据存储加载到缓存中。除了可以提高性能，还有助于保持缓存中保存的数据与底层数据存储中的数据之间的一致性。

### 背景和问题

应用程序使用缓存来改进对数据存储中保存的信息的重复访问。然而，期望缓存中的数据始终与数据存储中的数据完全一致是不切实际的。应用程序应实施有助于确保缓存中的数据尽可能保持最新状态的策略，但也可以检测并处理缓存中的数据过时的时候出现的情况。

### 解决方案

许多商业高速缓存系统提供直读和直写/后写操作(需加注释)。在这些系统中，应用程序通过引用缓存来检索数据。如果数据不在缓存中，则从数据存储中检索数据，并将其添加到缓存中。缓存中数据的任何修改也会自动写回数据存储区。

对于不提供此功能的缓存，使用缓存的应用程序负责维护数据。
应用程序可以通过实现缓存辅助策略来模拟通读缓存的功能。此策略将数据按需加载到缓存中。下图说明了使用Cache-Aside模式将数据存储在缓存中。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/cache-aside-diagram.png)

如果应用程序更新信息，则可以通过对数据存储进行修改，使缓存中的相应项目失效来执行直写策略。
当下一个项目需要时，使用缓存辅助策略更新数据存储中检索的数据并将其添加回缓存。

### 问题和注意事项

在决定如何实现此模式时，请考虑以下几点：

**缓存数据的生命周期**。 许多缓存都实现了过期策略让数据无效，如果在指定的时间段内没有访问，就将其从缓存中删除。为了使缓存有效，请确保过期策略与使用数据的应用程序的访问模式相匹配。过期期限不能太短，因为这可能导致应用程序不断从数据存储中检索数据并将其添加到缓存。同样，不要将过期期限长到缓存的数据过时。请记住，缓存对于相对静态的数据或频繁读取的数据是最有效的。


**驱逐数据**。 与数据来源的数据存储相比，大多数高速缓存的大小都有限，如果有需要它们会驱逐数据。大多数缓存采用最近最少使用的策略来选择要驱逐的缓存项，但也可能是可定制的。配置缓存的全局过期属性和其它属性以及每个缓存项的过期属性，以确保缓存具有成本效益。对每个缓存项应用全局驱逐策略并不总是合适的。例如，如果缓存项从数据存储中检索非常昂贵，将缓存项保持在缓存中更频繁的访问，成本低，收益高。

**预填缓存**。许多解决方案在缓存启动时会预填应用程序可能需要的缓存数据。如果某些数据过期或被驱逐，Cache-Aside模式仍然有用。

**一致性**。实现Cache-Aside模式不能保证数据存储和缓存之间的一致性。数据存储中的项目可以随时通过外部进程更改，下次加载项目时，此更改可能不会反映在缓存中。跨数据存储复制数据的系统中，如果频繁进行同步，一致性的问题可能会很严重。

**本地（内存中）缓存**。应用程序实例的本地可以有本地缓存，存储在内存中。如果应用程序重复访问相同的数据，本地缓存可能很有用。然而，本地缓存是私有的，因此不同的应用程序实例可能具有相同缓存数据的副本。缓存之间的数据可能会很快变的不一致，因此可能需要让保存在私有缓存中的数据过期，并更频繁地刷新数据。这些场景下请考虑调查共享或分布式缓存机制的使用。

### 何时使用该模式

在以下场景使用该模式：

* 缓存不提供原生的直读和直写操作。
* 资源需求是不可预知的。该模式使应用程序能够按需加载数据。它不会假设哪些数据应用程序会提前需要。

这种模式可能不适用于：

* 缓存数据集是静态的。如果数据可以放在可用的缓存空间中，在启动时用数据预填缓存，并应用防止数据过期的策略。
* 为Web集群中托管的Web应用程序缓存会话状态信息。在这种环境中，应该避免引入客户端-服务器亲和性的依赖关系。（？如何理解）

## 例子

在Microsoft Azure云，你可以使用Azure Redis Cache创建由应用程序的多个实例共享的分布式缓存。

要连接到Azure Redis Cache的实例，请调用静态`Connect`方法并传入连接字符串。该方法返回一个表示连接的`ConnectionMultiplexer`。在应用程序中共享`ConnectionMultiplexer`实例的一种方法是拥有返回连接实例的静态属性，和下面的例子类似。用线程安全的方式来初始化一个连接的实例。

```c#
private static ConnectionMultiplexer Connection;

// Redis Connection string info
private static Lazy<ConnectionMultiplexer> lazyConnection = new Lazy<ConnectionMultiplexer>(() =>
{
    string cacheConnection = ConfigurationManager.AppSettings["CacheConnection"].ToString();
    return ConnectionMultiplexer.Connect(cacheConnection);
});

public static ConnectionMultiplexer Connection => lazyConnection.Value;
```

下面代码中的`GetMyEntityAsync`方法展示了基于Azure Redis Cache的Cache-Aside模式的实现。该方法使用直读方式从缓存中检索对象。

通过使用整数`ID`作为key来标识对象。`GetMyEntityAsync`方法尝试从缓存中检索具有该key的项目。如果命中则返回。如果没有命中，`GetMyEntityAsync`方法将从数据存储中检索对象，将其添加到缓存中，然后返回。实际上从数据存储中读取数据的代码没有在这里，因为它数据存储相关。请注意，要配置缓存的项目过期，以防止其在其它地方更新时变得过时。

```c#
// Set five minute expiration as a default
private const double DefaultExpirationTimeInMinutes = 5.0;

public async Task<MyEntity> GetMyEntityAsync(int id)
{
  // Define a unique key for this method and its parameters.
  var key = $"MyEntity:{id}";
  var cache = Connection.GetDatabase();

  // Try to get the entity from the cache.
  var json = await cache.StringGetAsync(key).ConfigureAwait(false);
  var value = string.IsNullOrWhiteSpace(json) 
                ? default(MyEntity) 
                : JsonConvert.DeserializeObject<MyEntity>(json);

  if (value == null) // Cache miss
  {
    // If there's a cache miss, get the entity from the original store and cache it.
    // Code has been omitted because it's data store dependent.  
    value = ...;

    // Avoid caching a null value.
    if (value != null)
    {
      // Put the item in the cache with a custom expiration time that 
      // depends on how critical it is to have stale data.
      await cache.StringSetAsync(key, JsonConvert.SerializeObject(value)).ConfigureAwait(false);
      await cache.KeyExpireAsync(key, TimeSpan.FromMinutes(DefaultExpirationTimeInMinutes)).ConfigureAwait(false);
    }
  }

  return value;
}
```

> 这些例子使用Azure Redis Cache API来访问存储并从缓存中检索信息。更多内容请参阅[Microsoft Azure Redis Cache](https://docs.microsoft.com/en-us/azure/redis-cache/cache-dotnet-how-to-use-azure-redis-cache)以及[如何使用Redis Cache创建Web应用程序](https://docs.microsoft.com/en-us/azure/redis-cache/cache-web-app-howto)。

下面代码中的`UpdateEntityAsync`方法展示了在应用程序更改值时，如何让缓存中的对象无效。这是一个直写方法的例子。代码更新原始数据存储，然后通过调用`KeyDeleteAsync`方法（指定key）从缓存中删除缓存的数据。

>这个步骤的顺序很重要。如果在缓存更新之前删除数据，在数据存储中的数据更改之前，客户端应用程序有很短的时间来获取数据（因为数据不在高速缓存中），从而导致缓存包含陈旧的数据。


```c#
public async Task UpdateEntityAsync(MyEntity entity)
{
    // Invalidate the current cache object
    var cache = Connection.GetDatabase();
    var id = entity.Id;
    var key = $"MyEntity:{id}"; // Get the correct key for the cached object.
    await cache.KeyDeleteAsync(key).ConfigureAwait(false);

    // Update the object in the original data store
    await this.store.UpdateEntityAsync(entity).ConfigureAwait(false); 
}
```
### 相关指导

下面的内容可能和本模式的实现相关：

[缓存指导](https://docs.microsoft.com/en-us/azure/architecture/best-practices/caching)。介绍了有关如何在云解决方案中缓存数据的额外信息，以及实现缓存时应考虑的问题。

[数据一致性入门](https://msdn.microsoft.com/library/dn589800.aspx)。云应用程序通常使用分布在数据存储中的数据。 管理和维护数据一致性是系统的关键，特别是可能出现的并发和可用性问题。该入门介绍了有关分布式数据的一致性的问题，并总结了应用程序如何实现最终的一致性来维护数据的可用性。