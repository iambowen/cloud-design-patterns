# 索引表模式

通过查询经常引用的数据存储区中的字段创建索引。该模式可以通过允许应用程序更快速从数据存储中检索定位数据来提高查询性能。

## 背景和问题

许多数据存储使用主键来组织实体集合的数据。应用程序可以使用该键来定位和检索数据。下图是存储客户信息的数据存储的例子。主键是Customer ID。该图显示了由主键（Customer ID）组织的客户信息。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-1.png)

虽然主键对于根据该键的值查询获取数据很有价值，但如果需要根据某个其它字段检索数据，应用程序可能无法使用主键。在上面的例子中，只有通过引用某个其它属性（例如客户所在的town）的值来查询数据时，应用程序无法使用Customer ID主键来检索客户。要执行这样的查询，应用程序可能必须获取并检查每个客户记录，这可能是一个缓慢的过程。

许多关系数据库管理系统支持二级索引。二级索引是由一个或多个非主要（次要）键字段组织的单独的数据结构，指示存储每个索引值的数据的位置。二级索引中的项目通常按二级主键的值进行排序，以便能够快速查找数据。这些索引通常由数据库管理系统自动维护。

你可以创建尽可能多的二级索引，因为需要支持应用程序执行的不同查询。例如，在关系数据库中的Customers表中，Customer ID是主键，如果应用程序经常在其所在的town查找客户，在town字段中添加二级索引会有所帮助。

然而，虽然二级索引在关系数据库中很常见，但云应用程序提供的大多数NoSQL数据存储无法提供相同的功能。

## 解决方案

如果数据存储不支持二级索引，可以通过创建自己的索引表来手动模拟。索引表通过特定的键组织数据。通常使用三种策略来构造索引表，具体取决于所需的二级索引的数量以及应用程序执行的查询的性质。

第一个策略是复制每个索引表中的数据，但是通过不同的键进行组织（完全非规范化）。下图显示了通过Town和LastName组织相同客户信息的索引表。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-2.png)

该策略适于数据与使用每个键查询的次数相比是相对静态的场景。如果数据更具动态性，则维护每个索引表的处理开销变得太大，不适于使用。此外，如果数据量非常大，则存储重复数据所需的空间量很大。
第二种策略是创建由不同键组织的归一化索引表，并使用主键而不是复制原始数据，如下图所示。 原始数据称为事实表。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-3.png)

这种技术可节省空间并减少维护重复数据的开销。缺点是应用程序必须执行两次查找操作以使用二级键查找数据。必须在索引表中找到数据的主键，然后使用主键查找事实表中的数据。
第三种策略是创建通过重复频繁检索的字段的不同键组织的部分归一化索引表。参考事实表来访问访问频度较低的字段。下图显示了频繁访问的数据是如何复制到每个索引表的。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-4.png)

If an application frequently queries data by specifying a combination of values (for example, “Find all customers that live in Redmond and that have a last name of Smith”), you could implement the keys to the items in the index table as a concatenation of the Town attribute and the LastName attribute. The next figure shows an index table based on composite keys. The keys are sorted by Town, and then by LastName for records that have the same value for Town.
这一策略可以在前两种方法之间取得平衡。普通查询的数据可以通过使用单个查找快速检索，而空间和维护开销没有复制整个数据集大。

如果应用程序经常通过指定组合值来查询数据（例如，“查找居住在Redmond的所有客户，姓氏为Smith”），则可以将索引表中的项目的键实现为级联的Town属性和LastName属性。下图展示了基于复合键的索引表。键按Town排序，然后按Town值相同的LastName排序。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-5.png)
This can save the application from repeatedly calculating hash keys (an expensive operation) if it needs to retrieve data that falls within a range, or it needs to fetch data in order of the nonhashed key. For example, a query such as “Find all customers that live in Redmond” can be quickly resolved by locating the matching items in the index table, where they're all stored in a contiguous block. Then, follow the references to the customer data using the shard keys stored in the index table.
索引表可以加快对分片数据的查询操作，并且在碎片密钥散列时特别有用。下图中的例子分片键是Customer ID的哈希值。索引表可以通过非标记值（Town和LastName）组织数据，并提供散列分片键作为查找数据。如果需要检索落在一个范围内的数据，或者按照非散列键的顺序获取数据取，则可以避免应用程序重复计算散列键（昂贵的操作）。例如，可以通过在索引表中找到匹配的项目，将它们全部存储在连续的块中来快速解决诸如“查找居住在Redmond的所有客户”之类的查询。 然后，使用存储在索引表中的分片键，跟随客户数据的引用。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/index-table-figure-6.png)

## 问题和注意事项

Consider the following points when deciding how to implement this pattern:
The overhead of maintaining secondary indexes can be significant. You must analyze and understand the queries that your application uses. Only create index tables when they're likely to be used regularly. Don't create speculative index tables to support queries that an application doesn't perform, or performs only occasionally.
Duplicating data in an index table can add significant overhead in storage costs and the effort required to maintain multiple copies of data.
Implementing an index table as a normalized structure that references the original data requires an application to perform two lookup operations to find data. The first operation searches the index table to retrieve the primary key, and the second uses the primary key to fetch the data.
If a system incorporates a number of index tables over very large data sets, it can be difficult to maintain consistency between index tables and the original data. It might be possible to design the application around the eventual consistency model. For example, to insert, update, or delete data, an application could post a message to a queue and let a separate task perform the operation and maintain the index tables that reference this data asynchronously. For more information about implementing eventual consistency, see the Data Consistency Primer.
Microsoft Azure storage tables support transactional updates for changes made to data held in the same partition (referred to as entity group transactions). If you can store the data for a fact table and one or more index tables in the same partition, you can use this feature to help ensure consistency.
Index tables might themselves be partitioned or sharded.
When to use this pattern
Use this pattern to improve query performance when an application frequently needs to retrieve data by using a key other than the primary (or shard) key.
This pattern might not be useful when:
Data is volatile. An index table can become out of date very quickly, making it ineffective or making the overhead of maintaining the index table greater than any savings made by using it.
A field selected as the secondary key for an index table is nondiscriminating and can only have a small set of values (for example, gender).
The balance of the data values for a field selected as the secondary key for an index table are highly skewed. For example, if 90% of the records contain the same value in a field, then creating and maintaining an index table to look up data based on this field might create more overhead than scanning sequentially through the data. However, if queries very frequently target values that lie in the remaining 10%, this index can be useful. You should understand the queries that your application is performing, and how frequently they're performed.
Example
Azure storage tables provide a highly scalable key/value data store for applications running in the cloud. Applications store and retrieve data values by specifying a key. The data values can contain multiple fields, but the structure of a data item is opaque to table storage, which simply handles a data item as an array of bytes.
Azure storage tables also support sharding. The sharding key includes two elements, a partition key and a row key. Items that have the same partition key are stored in the same partition (shard), and the items are stored in row key order within a shard. Table storage is optimized for performing queries that fetch data falling within a contiguous range of row key values within a partition. If you're building cloud applications that store information in Azure tables, you should structure your data with this feature in mind.
For example, consider an application that stores information about movies. The application frequently queries movies by genre (action, documentary, historical, comedy, drama, and so on). You could create an Azure table with partitions for each genre by using the genre as the partition key, and specifying the movie name as the row key, as shown in the next figure.
Figure 7 - Movie data stored in an Azure table
This approach is less effective if the application also needs to query movies by starring actor. In this case, you can create a separate Azure table that acts as an index table. The partition key is the actor and the row key is the movie name. The data for each actor will be stored in separate partitions. If a movie stars more than one actor, the same movie will occur in multiple partitions.
You can duplicate the movie data in the values held by each partition by adopting the first approach described in the Solution section above. However, it's likely that each movie will be replicated several times (once for each actor), so it might be more efficient to partially denormalize the data to support the most common queries (such as the names of the other actors) and enable an application to retrieve any remaining details by including the partition key necessary to find the complete information in the genre partitions. This approach is described by the third option in the Solution section. The next figure shows this approach.
Figure 8 - Actor partitions acting as index tables for movie data
Related patterns and guidance
The following patterns and guidance might also be relevant when implementing this pattern:
Data Consistency Primer. An index table must be maintained as the data that it indexes changes. In the cloud, it might not be possible or appropriate to perform operations that update an index as part of the same transaction that modifies the data. In that case, an eventually consistent approach is more suitable. Provides information on the issues surrounding eventual consistency.
Sharding pattern. The Index Table pattern is frequently used in conjunction with data partitioned by using shards. The Sharding pattern provides more information on how to divide a data store into a set of shards.
Materialized View pattern. Instead of indexing data to support queries that summarize data, it might be more appropriate to create a materialized view of the data. Describes how to support efficient summary queries by generating prepopulated views over data.