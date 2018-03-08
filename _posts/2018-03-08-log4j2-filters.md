---
layout: post
title: "Log4j2 -- Filters"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 8. Filters

filter允许评估日志事件以确定是否或如何发布它们。Filter将在其过滤器方法之一上调用，并将返回一个结果，该结果是具有3个值之一的枚举值，即ACCEPT，DENY或NEUTRAL。

过滤器可以配置在以下四个位置：

- Context-wide的Filter直接在配置中配置。被这些filter拒绝的事件不会被传递给记录器进行进一步处理。一旦一个事件被一个Context-wide过滤器所接受，它将不会被任何其他Context-wide Filters所评估，也不会使用Logger的等级来过滤该事件。然而，事件将由Logger和Appender的Filters进行评估。
- Logger Filter在指定的记录器上进行配置。这些在Logger的Context-wide Filters和Log Level之后进行评估。被这些过滤器拒绝的事件将被丢弃，并且事件不会传递给父记录器，而不考虑可加性设置。
- Appender Filter用于确定特定的Appender是否应该处理事件的格式和发布。
- Appender Reference Filter用于确定Logger是否应将事件路由到appender。