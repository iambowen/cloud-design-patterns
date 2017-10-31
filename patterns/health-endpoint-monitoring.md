# 健康端点监控模式

在应用程序内部实现检查的功能，外部工具可以定期通过暴露出来的端点访问。这可以帮助验证应用程序和服务是否正常运行。

## 背景和问题

监控Web应用程序和后端服务是一个很好的实践，也是业务需求，以确保可用性和正常运行。但是，监控运行在云中的服务比本地服务更为困难。例如，没有对托管环境的完全控制权，并且服务通常依赖于平台供应商和其它服务。
影响云托管应用程序的因素很多，如网络延迟，底层计算和存储系统的性能和可用性以及它们之间的网络带宽。任何这些因素都可以导致服务完全或部分失败。因此，必须定期验证服务是否正常执行，以确保所需的可用性级别，这可能是你服务水平协议（SLA）的一部分。

## 解决方案

通过向应用程序的端点发送请求来实现健康监控。应用程序应执行必要的检查，并返回其状态的指示。
健康监测检查通常结合两个因素：
* 响应于健康验证端点的请求，应用程序或服务执行的检查（如果有）。
* 通过执行健康验证检查的工具或框架分析结果。
响应码反映应用程序、或者任何组件或者服务的状态。延迟或响应时间检查由监视工具或框架执行。下图概述了该模式。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/health-endpoint-monitoring-pattern.png)

应用程序中的健康检查代码可能执行的其它检查包括：

* 检查云存储或数据库的可用性和响应时间。
* 检查位于应用程序中的其它资源或服务，或应用程序使用的其它资源或服务。
* 相关的服务和工具通过向可配置的端点集发送请求来监视Web应用程序，并根据一组可配置规则评估结果。创建服务端点相对容易，其唯一目的是对系统执行一些功能测试。

监控工具可以执行的典型检查包括：

* 验证响应代码。例如，HTTP响应为200（OK）表示应用程序无错误地响应。监控系统还可以检查其他响应代码，以获得更全面的结果。
* 检查响应的内容以检测错误，即使返回200（OK）状态代码。这可以检测仅影响返回的网页或服务响应的一部分的错误。例如，返回页面的标题或查找指示正确页面的特定短语。
* 测量响应时间，即网络延迟与应用程序执行请求所花费的时间的组合。发现应用程序或网络正在出现问题带来的价值越多。
* 检查位于应用程序外部的资源或服务，例如应用程序用来全局缓存的内容分发网络（CDN）。
* 检查SSL证书是否到期。
* 测量应用程序URL的DNS查询的响应时间，以测量DNS延迟和DNS故障。
* 验证DNS查询返回的URL以确保入口正确。这可以避免通过攻击DNS服务成功引发的器恶意请求重定向。

可能的话，也可以从不同的本地或托管服务运行检查来测量和比较响应时间。理想情况下应该从靠近客户的位置监控应用程序，以便准确了解每个位置的性能。除了提供更强大的检查机制之外，还可以帮助决策应用程序的部署位置，以及是否将其部署到多个数据中心。
还应针对客户使用的所有服务实例运行测试，以确保应用程序正常运行。例如，如果客户的存储分布在多个存储帐户中，监控需要检查所有这些。

### 问题和注意事项

实现此模式时，请考虑以下几点：
如何验证响应。例如，一个200（OK）状态代码足以验证应用程序是否正常工作？虽然这提供了最基本的应用程序可用性的测量，并且是此模式的最小实现，但它提供了有关应用程序中的操作，趋势和可能的即将到来的问题的少量信息。
>确保应用程序仅在找到并正确处理目标资源时才返回200（OK）。在某些情况下，例如，使用主页来托管目标网页时，即使没有找到目标内容页面，服务器也会发回200（OK）状态代码而不是404（未找到）代码。
应用程序公开的端点数。一种方法是至少暴露应用使用的核心服务的一个端点，另一种用于低优先级服务，允许每个监控结果展示不同级别的重要性。还要考虑暴露更多的端点，例如每个核心服务的端点，以获得更多的监控粒度。例如，健康验证检查可能检查应用程序使用的数据库，存储和外部地理编码服务，每个服务需要的正常运行时间等级和响应时间都不相同。如果地理编码服务或其它一些后台任务几分钟不可用，应用程序仍然可以正常运行。

是否使用与通用访问相同的端点进行监控，但是是针对通用访问端点上的健康验证检查（例如/HealthCheck/{GUID}/）设计的特定路径。这允许应用程序中的一些功能测试由监控工具运行，例如添加新的用户注册，登录和订单测试，同时还验证一般访问端点是否可用。
响应监控请求在服务中收集的信息类型，以及如何返回此信息。大多数现有的工具和框架仅查看端点返回的HTTP状态代码。要返回并验证其它信息，可能需要创建自定义监视工具或服务。

要收集多少信息。在检查过程中执行过多的处理可能使应用程序过载，并影响其他用户。所需的时间可能超过监控系统的超时时间，因此它将应用程序标记为不可用。大多数应用程序包含了诸如错误处理程序和性能计数器之类的工具，在日志中记录性能和详细的错误信息，比从健康验证检查返回附加信息更加好用。

缓存端点状态。频繁运行健康检查成本太高。例如，如果通过仪表板报告运行状况，则不需要仪表板的每个请求来触发运行状况检查。而是定期检查系统运行状况并缓存状态。暴露返回缓存状态的端点。

如何配置监控端点的安全性以保护其免受公共访问，这可能会使应用程序暴露于恶意攻击，存在敏感信息的暴露或吸引拒绝服务（DoS）攻击。通常这应该在应用程序配置中完成，以便可以轻松更新，而无需重新启动应用程序。考虑使用以下一种或多种技术：

* 通过要求身份验证来保护端点。可以通过在请求头中使用身份验证安全密钥或通过传递带有请求的凭据来进行此操作，前提是监控服务或工具支持身份验证。
   * 使用模糊或隐藏的端点。例如，将端点暴露于与默认应用程序URL使用的不同IP地址上的端点，在非标准HTTP端口上配置端点或使用测试页面的复杂路径。通常可以在应用程序配置中指定其它端点地址和端口，如果需要，可以将这些端点的条目添加到DNS服务器，以避免直接指定IP地址。
   * 在接受诸如键值对或操作模式值的参数的端点上公开一种方法。根据为此参数提供的值，在接收到请求时，代码可以执行特定的测试或一组测试，或者如果未识别参数值，则返回404错误。识别的参数值可以在应用程序配置中设置。
   >DoS攻击可能对执行基本功能测试的单独端点的影响较小，而不会影响应用程序的操作。理想情况下，避免使用可能暴露敏感信息的测试。如果您必须返回可能对攻击者有用的信息，请考虑如何保护端点和数据免遭未经授权的访问。在这种情况下，只是依赖于晦涩是不够的。您还应考虑使用HTTPS连接和加密任何敏感数据，尽管这将增加服务器上的负载。

* 如何访问使用身份验证保护的端点。并不是所有的工具和框架都可以配置为包含身份验证请求的凭据。例如，Microsoft Azure内置的健康验证功能无法提供身份验证凭据。可以使用一些第三方的服务如Pingdom，Panopta，NewRelic和Statuscake。
* 如何确保监控代理正常运行。一种方法是公开一个简单地从应用程序配置返回值的端点或可用于测试代理的随机值。
>还要确保监控系统对自身进行检查，如自检和内置测试，以避免发生假阳性结果。

## 何时使用该模式

此模式适用于以下场景：
* 监控网站和网络应用程序以验证可用性。
* 监控网站和网络应用程序以检查正确的操作。
* 监控中间层或共享服务，以检测和隔离可能会中断其它应用程序的故障。
* 补充应用程序中的现有仪器，如性能计数器和错误处理程序。健康检查不会取代应用程序中记录和审核的要求。仪器可以为现有框架提供有价值的信息，监控计数器和错误日志可以用来检测故障或其它问题。但是，如果应用程序不可用，则无法提供信息。

## 案例

下面的例子中`HealthCheckController`类代码（代码可在[GitHub](https://github.com/mspnp/cloud-design-patterns/tree/master/health-endpoint-monitoring)上）展示了一个端点，用于执行一系列健康检查。
`CoreServices`方法对应用程序中使用的服务执行一系列检查。如果所有的测试运行没有错误，方法返回200（OK）状态代码。如果测试引发异常，方法将返回一个500（内部错误）状态代码。如果监控工具或框架能使用该方法，可以选择在发生错误时返回附加信息。

```java
public ActionResult CoreServices()
{
  try
  {
    // Run a simple check to ensure the database is available.
    DataStore.Instance.CoreHealthCheck();

    // Run a simple check on our external service.
    MyExternalService.Instance.CoreHealthCheck();
  }
  catch (Exception ex)
  {
    Trace.TraceError("Exception in basic health check: {0}", ex.Message);

    // This can optionally return different status codes based on the exception.
    // Optionally it could return more details about the exception.
    // The additional information could be used by administrators who access the
    // endpoint with a browser, or using a ping utility that can display the
    // additional information.
    return new HttpStatusCodeResult((int)HttpStatusCode.InternalServerError);
  }
  return new HttpStatusCodeResult((int)HttpStatusCode.OK);
}
```

`ObscurePath`方法从应用程序配置读取路径并将其用作测试的端点。这个C＃的例子也显示了如何接受ID作为参数，并用它来检查有效的请求。
```java
public ActionResult ObscurePath(string id)
{
  // The id could be used as a simple way to obscure or hide the endpoint.
  // The id to match could be retrieved from configuration and, if matched,
  // perform a specific set of tests and return the result. If not matched it
  // could return a 404 (Not Found) status.

  // The obscure path can be set through configuration to hide the endpoint.
  var hiddenPathKey = CloudConfigurationManager.GetSetting("Test.ObscurePath");

  // If the value passed does not match that in configuration, return 404 (Not Found).
  if (!string.Equals(id, hiddenPathKey))
  {
    return new HttpStatusCodeResult((int)HttpStatusCode.NotFound);
  }

  // Else continue and run the tests...
  // Return results from the core services test.
  return this.CoreServices();
}
```
`TestResponseFromConfig`方法显示如何暴露一个检查指定配置设置值的端点。

```java
public ActionResult TestResponseFromConfig()
{
  // Health check that returns a response code set in configuration for testing.
  var returnStatusCodeSetting = CloudConfigurationManager.GetSetting(
                                                          "Test.ReturnStatusCode");

  int returnStatusCode;

  if (!int.TryParse(returnStatusCodeSetting, out returnStatusCode))
  {
    returnStatusCode = (int)HttpStatusCode.OK;
  }

  return new HttpStatusCodeResult(returnStatusCode);
}
```
## 监控Azure托管应用程序中的端点

Azure应用程序中监视端点的选项有：
* 使用Azure的内置监控功能。
* 使用第三方服务或框架，如Microsoft System Center Operations Manager。
* 创建一个自定义实用程序或一个在自己的或托管的服务器上运行的服务。

>即使Azure提供了相当全面的监控选项，也可以使用其它服务和工具来提供额外的信息。Azure管理服务提供内置的警报规则监控机制。 Azure门户中的管理服务页面的告警部分可以为服务配置最多10个警报规则。这些规则包含（如CPU负载）或每秒请求或错误数量指定条件和阈值，并且服务可以自动向每个规则中定义的地址发送电子邮件通知。

取决于应用程序的托管机制（如Web站点，云服务，虚拟机或移动服务），监控条件各不相同，但这些都拥有在服务的设置中指定创建使用Web端点的警报规则的功能。端点应及时响应，以使警报系统能够检测到应用程序正常运行。
阅读更多有关[创建警报](https://azure.microsoft.com/documentation/articles/insights-receive-alert-notifications/)通知的信息。

如果你在Azure Cloud Services Web和工作角色或虚拟机中托管应用程序，可以利用Azure中的一种内置服务（称为流量管理器）。流量管理器是路由和负载平衡服务，可以根据一系列规则和设置将请求分发到云服务托管应用程序的特定实例。
除了路由请求之外，流量管理器会定期ping指定的URL，端口和相对路径，以确定其规则中定义的应用程序的哪些实例是活动的并且响应请求。如果检测到状态代码200（OK），它将应用程序标记为可用。任何其它状态代码都会导致流量管理器将应用程序标记为离线。你可以在流量管理器控制台中查看状态，并配置规则将请求重新路由到正在响应的应用程序的其它实例。
但是，流量管理器只会等待十秒钟从监控URL接收到响应。因此，应该确保在此时间内执行健康验证码，从而允许从流量管理器往返应用程序的网络延迟。
>阅读更多有关[使用流量管理器监控应用程序](https://azure.microsoft.com/documentation/services/traffic-manager/)的内容。[多数据中心部署指南](https://msdn.microsoft.com/library/dn589779.aspx)中还讨论了流量管理器。

## 相关指南
以下指南在实现此模式时有用：
* [仪表与遥测指南](https://msdn.microsoft.com/library/dn589775.aspx)。检查服务和组件的运行状况通常通过探测来完成，但是也可以使用信息来监控应用程序性能并检测运行时发生的事件。该数据可以作为健康监测的附加信息传回监控工具。仪器和遥测指南探索收集仪器在应用程序中收集的远程诊断信息。
* [接收警报通知](https://azure.microsoft.com/documentation/articles/insights-receive-alert-notifications/)。
* 使用此模式的[示例应用程序下载](https://github.com/mspnp/cloud-design-patterns/tree/master/health-endpoint-monitoring)。