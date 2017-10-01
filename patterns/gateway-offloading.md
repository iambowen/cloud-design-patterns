# 网关卸载模式

将共享的或专门的服务功能卸载到一个网关代理中去。通过将共享的服务功能从应用其它部分转移到网关里，这种模式可以简化应用的开发，例如SSL证书的使用。

## 背景与问题

某些特性往往是跨多个服务公用的，这些特性通常需要配置、管理和维护。散布于每个应用部署中的共享的或专门的服务会增加管理开销，并提高部署出错的可能性。任何对于共享特性的更新都必须对所有共享该特性的服务进行部署。

正确地处理安全问题（令牌校验、加密、SSL证书管理）和其它复杂任务需要团队成员具有高度的专业技能。例如，某个应用所需的证书必须在所有应用实例上进行配置和部署。每当有新的部署，就必须管理证书以确保其不会过期。在每次应用部署中，任何即将过期的公共证书必须得到更新、测试和验证。

在大规模部署下，其它公共服务，比如身份认证、鉴权、日志、监控或限流可能会变得难于实现和管理。最好将这类功能统一起来，以减少开销和出错机会。

## 解决方案

将某些特性卸载到一个API网关中去，特别是那些横切关注点，比如证书管理、身份认证、SSL终端、监控、协议转译或限流。

下图中显示了一个用于终止入站SSL连接的API网关。它代表原始请求者向网关的任意HTTP上游服务器请求数据。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gateway-offload.png)

该模式的好处在于：

* 简化了服务的开发，去除了所需资源的分发和维护需求，比如对于安全站点所需的web服务器证书和配置。更简化的配置，使得管理和扩展更为容易，并使得服务升级更为容易。

* 允许专门的团队来实现那些需要专业技能的特性，比如安全。这也让你的核心团队能够专注于应用功能，而将这些专业的横切关注点交给相关专家。

* 提供请求与响应在日志和监控上的某种一致性。即使某个服务未能被正确安装，仍能够通过配置的网关保证提供最小级别的监控和日志。

## 问题与注意事项

* 确保API网关是高可用的，而且能够从故障中恢复。你的API网关应当运行多个实例以避免单点故障。
* 确保网关在设计时已考虑到应用和终端的容量与扩展需求。确保网关不会成为应用的瓶颈，能够充分扩展。
* 只卸载那些为整个应用所公用的特性，比如安全或数据传输。
* 永远不要把业务逻辑卸载给API网关。
* 如果需要跟踪事务，为记录日志，可以考虑生成相互关联的ID。

## 何时使用该模式

在以下场景使用该模式：

* 应用的部署具有共享关注点，比如SSL证书或者加密。
* 在整个应用的不同部分进行部署时，含有共同的特性，却具有不同的资源需求，例如内存资源、存储容量或网络连接。
* 希望将某些问题的职责交给更为专业的团队，比如网络安全、限流或其它网络边界关注点。

如果会引入服务间的耦合，则这种模式可能就不太合适。

## 例子

如下配置将Nginx作为SSL卸载工具，终止一个入站SSL连接，并将连接分发给三个上游HTTP服务器之一。

```
upstream iis {
        server  10.3.0.10    max_fails=3    fail_timeout=15s;
        server  10.3.0.20    max_fails=3    fail_timeout=15s;
        server  10.3.0.30    max_fails=3    fail_timeout=15s;
}

server {
        listen 443;
        ssl on;
        ssl_certificate /etc/nginx/ssl/domain.cer;
        ssl_certificate_key /etc/nginx/ssl/domain.key;

        location / {
                set $targ iis;
                proxy_pass http://$targ;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto https;
proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header Host $host;
        }
}
```

## 相关指导

* [前端专属的后端模式](patterns/backends-for-frontends.md)
* [网关聚合模式](patterns/gateway-aggregation.md)
* [网关路由模式](patterns/gateway-routing.md)
