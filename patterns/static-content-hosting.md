# 静态内容托管模式

将静态内容部署到云存储服务，直接返回给浏览器。这可以减少使用计算实例的昂贵成本。

## 背景和问题

Web应用程序通常包含静态内容的一些元素。这些静态内容可能包括HTML页面和浏览器所需的其它资源（如图像和文档），作为HTML页面的一部分（如内联图像，样式表和客户端JavaScript文件）或单独下载（ 如PDF文件）。

虽然Web服务器通过高效的动态页面代码执行和输出缓存来优化请求，但仍然需要处理下载静态内容的请求。这消耗了能更有效利用的处理循环。

## 解决方案

在大多数云托管环境中，通过将某些应用程序的资源和静态页面部署在存储服务中，可以最大限度地减少对计算实例的需求（例如，使用较小的实例或更少的实例）。云托管存储的成本通常远少于计算实例的成本。

当在存储服务中托管应用程序的某些部分时，主要注意事项与应用程序的部署以及保护匿名用户无法使用的资源相关。

## 问题和注意事项

在考虑如何实现此模式时，请考虑以下几点：

* 托管存储服务必须公开一个用户可以访问的HTTP端点来下载静态资源。一些存储服务还支持HTTPS，为需要SSL的存储服务中托管资源。
* 为了获得最佳性能和可用性，请考虑使用CDN将存储容器的内容缓存在世界各地的多个数据中心中。但是使用CDN需要额外的开支。
* 默认情况下，存储帐户经常在不同区域复制，以提供数据中心外更高的可用性。这意味着IP地址可能会更改，但URL将保持不变。
* 当某些内容位于存储帐户而其它内容位于托管的计算实例中时，应用程序部署和更新变得更具挑战。可能需要执行单独的部署，并版本化应用程序和内容以方便管理，尤其是静态内容包含脚本文件或UI组件时。但是如果只更新静态资源，则只需将其上传到存储帐户，而不需要重新部署应用程序。
* 存储服务可能不支持使用自定义域名。在这种情况下，有必要在链接中指定资源的完整URL，因为它们与动态生成的包含链接的内容在不同的域中。
* 必须为存储容器配置公开读取访问权限，但确保不要公开写入权限以防止用户能够上传内容。考虑使用代客钥匙或令牌来控制不对匿名用户可用资源的访问 - 更多相关信息，请参阅[代客钥匙模式](valet-key.html)。

## 何时使用该模式

在以下场景使用该模式：

* 最小化包含一些静态资源的网站和应用程序的托管成本。
* 最小化仅由静态内容和资源组成的网站的托管费用。根据托管服务提供商的存储系统的功能，可以在一个存储帐户中完全托管一个完全静态的网站。
* 为运行在其它托管环境或本地服务器的应用程序暴露静态资源和内容。
* 使用内容分发网络在多个地理区域中定位内容，CDN将存储帐户的内容缓存在世界各地的多个数据中心内。
* 监控成本和带宽使用。对于某些或所有静态内容使用单独的存储帐户，可以更容易地将托管成本和运行成本分开。

此模式在以下场景用处不大：

* 应用程序需要在将其传递给客户端之前对静态内容执行一些处理。例如，可能需要向文档添加时间戳。
* 静态内容很小。从单独的存储中检索此内容的开销可能超过将其从计算资源中分离出来的成本。

## 案例

位于Azure Blob存储中的静态内容可以通过Web浏览器直接访问。Azure提供了可以公开暴露给浏览器的基于HTTP的存储接口。例如，Azure Blob存储容器中的内容使用以下形式的URL：
```
http：// [storage-account-name] .blob.core.windows.net / [container-name] / [file-name]
```
上传内容时，需要创建一个或多个blob容器来保存文件和文档。请注意，新容器的默认权限为`Private`，您必须将其更改为`Public`以允许客户端访问内容。如果有必要保护内容免受匿名访问，可以实现[代客钥匙模式](valet-key.html)，用户必须提供有效的令牌来下载资源。
> [Blob服务概念](https://msdn.microsoft.com/library/azure/dd179376.aspx)介绍了关于blob存储的信息，以及访问和使用方式。

每个页面中的链接将指定资源的URL，客户端将直接从存储服务访问它。该图说明了直接从存储服务提供应用程序的静态部分。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/static-content-hosting-pattern.png)
浏览器页面中的链接必须指定blob容器和资源的完整URL。例如，包含指向公共容器中的图像的链接的页面可能包含以下HTML内容。
```html
<img src="http://mystorageaccount.blob.core.windows.net/myresources/image1.png"
     alt="My image" />
```

> 如果通过使用代客钥匙，如Azure共享访问签名，来保护资源，该签名必须包含在链接中的URL中。

名为StaticContentHosting的解决方案，演示了如何使用外部存储进行静态资源，可以在[GitHub](https://github.com/mspnp/cloud-design-patterns/tree/master/static-content-hosting)上获得。 StaticContentHosting.Cloud项目包含指定保存静态内容的存储帐户和容器的配置文件。

```xml
<Setting name="StaticContent.StorageConnectionString"
         value="UseDevelopmentStorage=true" />
<Setting name="StaticContent.Container" value="static-content" />
```
StaticContentHosting.Web工程的`Settings.cs`文件中的`Settings`类包含提取这些值并构建包含云存储帐户容器URL的字符串值的方法。

```java
public class Settings
{
  public static string StaticContentStorageConnectionString {
    get
    {
      return RoleEnvironment.GetConfigurationSettingValue(
                              "StaticContent.StorageConnectionString");
    }
  }

  public static string StaticContentContainer
  {
    get
    {
      return RoleEnvironment.GetConfigurationSettingValue("StaticContent.Container");
    }
  }

  public static string StaticContentBaseUrl
  {
    get
    {
      var account = CloudStorageAccount.Parse(StaticContentStorageConnectionString);

      return string.Format("{0}/{1}", account.BlobEndpoint.ToString().TrimEnd('/'),
                                      StaticContentContainer.TrimStart('/'));
    }
  }
}
```

StaticContentUrlHtmlHelper.cs文件中的`StaticContentUrlHtmlHelper`类有一个名为`StaticContentUrl`的公有方法，如果传递给它的URL以ASP.NET根路径字符（〜）开头，该方法生成包含云存储帐户路径的URL。
```java
public static class StaticContentUrlHtmlHelper
{
  public static string StaticContentUrl(this HtmlHelper helper, string contentPath)
  {
    if (contentPath.StartsWith("~"))
    {
      contentPath = contentPath.Substring(1);
    }

    contentPath = string.Format("{0}/{1}", Settings.StaticContentBaseUrl.TrimEnd('/'),
                                contentPath.TrimStart('/'));

    var url = new UrlHelper(helper.ViewContext.RequestContext);

    return url.Content(contentPath);
  }
}
```

Views\Home目录中的`Index.cs`html文件包含一个使用`StaticContentUrl`方法为其src属性创建URL的图像元素。
```html
<img src="@Html.StaticContentUrl("~/media/orderedList1.png")" alt="Test Image" />
```

## 相关模式和指南

* 在[GitHub](https://github.com/mspnp/cloud-design-patterns/tree/master/static-content-hosting)上可以找到此模式的例子。
* [随从钥匙](valet-key.md)模式。如果目标资源需要对匿名用户不可用，则有必要在保存静态内容的存储上实现安全性。描述如何使用向客户端提供受限直接访问特定资源或服务（如云托管存储服务）的令牌或密钥。
* 在Infosys博客上关于[在Azure上部署静态网站的有效方式](http://www.infosysblogs.com/microsoft/2010/06/an_efficient_way_of_deploying.html)的文章。
* [Blob服务概念](https://msdn.microsoft.com/library/azure/dd179376.aspx)