### 云设计模式

这些设计模式对于在云中构建可靠，可扩展，安全的应用程序非常有用。
书中的每个模式描述了要解决的问题，使用模式的注意事项以及基于Microsoft Azure的例子。大多数模式包括如何在Azure上实现模式的代码示例或代码段。但是，大多数模式都适用于分布式系统，和在那个云平台托管没有关系。

## 开发云上应用面临的挑战

### 可用性
可用性是系统正常工作时间的比例，通常以运行时间的百分比来衡量。它会受到系统错误，基础设施问题，恶意攻击和系统负载的影响。云应用程序通常会为用户提供服务等级协议（SLA），因此应用程序必须设计为最大限度地提高可用性。

### 数据管理

数据管理是云应用的关键要素，影响了大多数的质量属性。因为性能，可扩展性或可用性等原因，数据通常托管在不同的位置并跨多个服务器，这可能会带来一系列挑战。例如，必须保持数据一致性，并且数据通常需要在不同位置间同步。

### 设计和实现

良好的设计涵盖了如组件设计和部署中的一致性和相关性，简化管理和开发的可维护性，以及允许组件和子系统在其它应用程序和场景中的可重用性等因素。在设计和实施阶段做出的决策对托管在云商的应用程序和服务的质量、总体拥有成本产生巨大的影响。
 
### 消息

云应用程序的分布式特性需要一种连接组件和服务的消息传递基础设施，理想情况下以松散耦合的方式，以便最大限度地提高可扩展性。 异步消息系统使用广泛，提供了许多好处，但也带来了诸如消息排序，[毒药消息管理](https://docs.microsoft.com/en-us/dotnet/framework/wcf/feature-details/poison-message-handling)，幂等等的挑战

### 管理和监控

云应用程序运行在基础设施甚至操作系统无法完全控制的远程数据中心。管理和监控比本地部署更困难。应用程序必须公开管理员和操作员可以使用的运行时信息来管理和监视系统，以及支持不断变化的业务需求和定制，而不需要停止或重新部署应用程序。

Cloud Design Patterns
 
                Catalog of patterns
PATTERN SUMMARY
      Ambassador
  Create helper services that send network requests on behalf of a consumer service or application.
   Anti-Corruption Layer
  Implement a façade or adapter layer between a modern application and a legacy system.
   Backends for Frontends
  Create separate backend services to be consumed by specific frontend applications or interfaces.
   Bulkhead
  Isolate elements of an application into pools so that if one fails, the others will continue to function.
   Cache-Aside
   Load data on demand into a cache from a data store
   Circuit Breaker
 Handle faults that might take a variable amount of time to fix when connecting to a remote service or resource.
   CQRS
  Segregate operations that read data from operations that update data by using separate interfaces.
   Compensating Transaction
  Undo the work performed by a series of steps, which together define an eventually consistent operation.
   Competing Consumers
   Enable multiple concurrent consumers to process messages received on the same messaging channel.
     Performance and Scalability
Performance is an indication of the responsiveness of a system to execute any action within a given time interval, while scalability is ability of a system either to handle increases in load without impact on performance or for the available resources to be readily increased. Cloud applications typically encounter variable workloads and peaks in activity. Predicting these, especially in a multi-tenant scenario, is almost impossible. Instead, applications should be able to scale out within limits to meet peaks in demand, and scale in when demand decreases. Scalability concerns not just compute instances, but other elements such as data storage, messaging infrastructure, and more.
      Resiliency
Resiliency is the ability of a system to gracefully handle and recover from failures. The nature of cloud hosting, where applications are often multi-tenant, use shared platform services, compete for resources and bandwidth, communicate over the Internet, and run on commodity hardware means there is an increased likelihood that both transient and more permanent faults will arise. Detecting failures, and recovering quickly and efficiently, is necessary to maintain resiliency.
      Security
Security is the capability of a system to prevent malicious or accidental actions outside of the designed usage, and to prevent disclosure or loss of information. Cloud applications are exposed on the Internet outside trusted on- premises boundaries, are often open to the public, and may serve untrusted users. Applications must be designed and deployed in a way that protects them from malicious attacks, restricts access to only approved users, and protects sensitive data.
    
                    PATTERN
SUMMARY
     Compute Resource Consolidation
Consolidate multiple tasks or operations into a single computational unit
     Event Sourcing
Use an append-only store to record the full series of events that describe actions taken on data in a domain.
     External Configuration Store
Move configuration information out of the application deployment package to a centralized location.
     Federated Identity
Delegate authentication to an external identity provider.
     Gatekeeper
Protect applications and services by using a dedicated host instance that acts as a broker between clients and the application or service, validates and sanitizes requests, and passes requests and data between them.
     Gateway Aggregation
Use a gateway to aggregate multiple individual requests into a single request.
     Gateway Offloading
Offload shared or specialized service functionality to a gateway proxy.
     Gateway Routing
Route requests to multiple services using a single endpoint.
     Health Endpoint Monitoring
Implement functional checks in an application that external tools can access through exposed endpoints at regular intervals.
     Index Table
Create indexes over the fields in data stores that are frequently referenced by queries.
     Leader Election
Coordinate the actions performed by a collection of collaborating task instances in a distributed application by electing one instance as the leader that assumes responsibility for managing the other instances.
     Materialized View
Generate prepopulated views over the data in one or more data stores when the data isn't ideally formatted for required query operations.
     Pipes and Filters
Break down a task that performs complex processing into a series of separate elements that can be reused.
     Priority Queue
Prioritize requests sent to services so that requests with a higher priority are received and processed more quickly than those with a lower priority.
     Queue-Based Load Leveling
Use a queue that acts as a buffer between a task and a service that it invokes in order to smooth intermittent heavy loads.
     Retry
Enable an application to handle anticipated, temporary failures when it tries to connect to a service or network resource by transparently retrying an operation that's previously failed.
  
       
   PATTERN
SUMMARY
     Scheduler Agent Supervisor
Coordinate a set of actions across a distributed set of services and other remote resources.
     Sharding
Divide a data store into a set of horizontal partitions or shards.
     Sidecar
Deploy components of an application into a separate process or container to provide isolation and encapsulation.
     Static Content Hosting
Deploy static content to a cloud-based storage service that can deliver them directly to the client.
     Strangler
Incrementally migrate a legacy system by gradually replacing specific pieces of functionality with new applications and services.
     Throttling
Control the consumption of resources used by an instance of an application, an individual tenant, or an entire service.
     

