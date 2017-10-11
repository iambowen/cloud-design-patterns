# 网关聚合模式

使用网关将多个独立请求聚合为一个单独的请求。当客户端必须向不同的后端系统发起多次调用才能完成某个操作时，该模式十分有用。

## 背景与问题

要执行某个任务时，客户端必须向不同的后端服务发起多次调用。依赖于多个服务的应用在执行任务时，必须为每个请求都扩展资源。当新的特性或服务添加到该应用时，需要添加额外的请求，而且增加了资源需求与网络调用。这些在客户端与后端之间的通信量会对应用的性能和可扩展性造成不利影响。微服务架构下这种情况更为普遍，因为应用是由若干个更小的服务构成的，天然地需要更多数量的跨服务调用。

在下图中，客户端向服务发送请求（1，2，3）。每个服务处理这些请求并向应用返回响应（4，5，6）。在高延迟的蜂窝网络中，以这种方式使用独立的请求无疑是低效的，而且可能出现连接断开或未完成的请求。各个请求还有可能并行执行，应用就必须为每个请求发送、等待和处理数据，这些都是在各自独立的连接中的，增加了失败的可能性。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gateway-aggregation-problem.png)

## 解决方案

使用网关来减少客户端与服务之间的通信量。网关接收客户请求，将请求分发到不同的后端系统，然后将结果聚合后返回给请求的客户端。

这种模式可以减少应用向后端服务发送请求的数量，并且改善应用在高延迟网络中的性能。

在下图中，应用向网关发送一个请求（1）。该请求包括了一系列额外请求组成的包。网关将该请求包分解为不同的请求，并发送给各自的相关服务（2）。每个服务向网关返回一个相应（3）。网关将来自各个服务的响应组合为一个响应后返回给应用（4）。应用只需向网关发送一个单独的请求，并从网关得到一个单独的响应。

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gateway-aggregation.png)

## 问题与注意事项

* 网关不应引入后端服务之间的耦合。
* 网关应该位于后端服务附近，以尽可能地减少之间的延迟。
* 网关服务可能引入单点故障。确保网关设计的合理性，以满足你的应用的可用性要求。
* 网关有可能引入瓶颈。确保网关有足够的性能来处理负载，并能够被扩展以满足预期的增长。
* 对网关进行负载测试，以确保你不会为服务引入级联性故障。
* 使用隔板、断路器、重试、超时等技术实现弹性设计。
* 如果一个或多个服务调用占用时间过长，对其超时并只返回一部分数据也是可以接受的。需要考虑你的应用如何处理这种场景。
* 使用异步I/O以确保后端的延时不会引起应用的性能问题。
* 使用相互关联的ID来实现分布式跟踪，以跟踪到各个单独请求。
* 监控请求度量指标和响应大小。
* 考虑将缓存数据作为一种处理故障的故障转移策略。
* 考虑在网关后面放一个聚合服务，而不是将聚合做入到网关里。因为请求聚合与网关中的其他服务相比可能会有不同的资源需求，因此有可能影响到网关的路由和卸载功能。

## 何时使用该模式

在以下场景使用该模式：

* 客户端需要与多个后端服务通信才能完成某个操作时。
* 客户端可能使用具有显著延迟的网络，比如蜂窝网络。

这种模式可能不适用于：

* 在完成多个操作时，希望减少某个客户端与某个服务之间的调用数量时。这种场景下，可能更适于为该服务添加一个批量操作。
* 客户端或者应用位于后端服务附近，且网络延迟不是非常显著时。


## 例子

以下例子展示了如何在NGINX服务上使用Lua创建一个简单的网关聚合。

```lua
worker_processes  4;

events {
  worker_connections 1024;
}

http {
  server {
    listen 80;

    location = /batch {
      content_by_lua '
        ngx.req.read_body()

        -- read json body content
        local cjson = require "cjson"
        local batch = cjson.decode(ngx.req.get_body_data())["batch"]

        -- create capture_multi table
        local requests = {}
        for i, item in ipairs(batch) do
          table.insert(requests, {item.relative_url, { method = ngx.HTTP_GET}})
        end

        -- execute batch requests in parallel
        local results = {}
        local resps = { ngx.location.capture_multi(requests) }
        for i, res in ipairs(resps) do
          table.insert(results, {status = res.status, body = cjson.decode(res.body), header = res.header})
        end

        ngx.say(cjson.encode({results = results}))
      ';
    }

    location = /service1 {
      default_type application/json;
      echo '{"attr1":"val1"}';
    }

    location = /service2 {
      default_type application/json;
      echo '{"attr2":"val2"}';
    }
  }
}
```

## 相关指南

* [前端专属的后端模式](patterns/backends-for-frontends.md)
* [网关卸载模式](patterns/gateway-offloading.md)
* [网关路由模式](patterns/gateway-routing.md)