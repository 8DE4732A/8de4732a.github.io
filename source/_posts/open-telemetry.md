---
title: open telemetry
date: 2022-07-11 10:17:22
tags:
---
# ****Observability****

[traces](https://opentelemetry.io/docs/concepts/observability-primer/#distributed-traces), [metrics](https://opentelemetry.io/docs/concepts/observability-primer/#reliability--metrics), [logs](https://opentelemetry.io/docs/concepts/observability-primer/#logs)

Metrics是在一段时间内组成单个逻辑测量、计数器或直方图的原子，例如，服务调用的QPS、响应时间、错误请求发生率，目的是为了创建集中式度量系统，侧重于技术指标的收集与观测;Logging用于记录离散的事件，例如，应用程序的调试信息或错误信息，目的是搭建集中式日志系统，侧重于统一采集、存储与检索各个微服务的日志;Tracing处理请求范围内的信息，例如，一次远程方法调用的执行过程和耗时，目的是形成分布式追踪系统，侧重于串联请求在微服务间的调用情况、继而进行追踪与APM分析。

![observability](assets/observability.png)

traces: jaeger zipKin

metrics: prometheus

logging: ELK

**[OpenTelemetry](https://opentelemetry.io/docs/concepts/what-is-opentelemetry)** is the mechanism by which application code is instrumented, to help make a system observable.

# Java Agent

[javaagent使用指南](https://www.cnblogs.com/rickiyang/p/11368932.html)

[探秘 Java 热部署三（Java agent agentmain）](https://www.cnblogs.com/stateis0/p/9062201.html)

[JVM源码分析之javaagent原理完全解读_Java_李嘉鹏_InfoQ精选文章](https://www.infoq.cn/article/javaagent-illustrated/)

[OpenTelemetry 可观测建设](https://zhuanlan.zhihu.com/p/522054923)

**trace 是请求在分布式系统中的整个链路视图，span 则代表整个链路中不同服务内部的视图，span 组合在一起就是整个 trace 的视图。**

# Source Code

`ThreadLocalContextStorage` 基于ThreadLocal的上下文存储

instrumentation以SPI机制加载

## `java-http-client` 为例

```jsx
static {
    SETTER = new HttpHeadersSetter(GlobalOpenTelemetry.getPropagators());

    JdkHttpAttributesGetter httpAttributesGetter = new JdkHttpAttributesGetter();
    JdkHttpNetAttributesGetter netAttributesGetter = new JdkHttpNetAttributesGetter();

    INSTRUMENTER =
        Instrumenter.<HttpRequest, HttpResponse<?>>builder(
                GlobalOpenTelemetry.get(),
                "io.opentelemetry.java-http-client",
                HttpSpanNameExtractor.create(httpAttributesGetter))
            .setSpanStatusExtractor(HttpSpanStatusExtractor.create(httpAttributesGetter))
            .addAttributesExtractor(HttpClientAttributesExtractor.create(httpAttributesGetter))
            .addAttributesExtractor(NetClientAttributesExtractor.create(netAttributesGetter))
            .addAttributesExtractor(PeerServiceAttributesExtractor.create(netAttributesGetter))
            .addOperationMetrics(HttpClientMetrics.get())
            .newClientInstrumenter(SETTER);
  }
```

clientInsturmenter通过 http header实现

![open-telemetry-http](assets/open-telemetry-http.png)