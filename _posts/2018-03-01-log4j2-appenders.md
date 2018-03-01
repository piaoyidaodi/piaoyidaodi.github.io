---
layout: post
title: "Log4j2 -- Appenders"
categories: Log4j2
tag: Java-Libs
---
> `Log4j2-v2.10.0-User's Guide`

## 7. Appenders

Appender负责将LogEvents交付到目的地。每个Appender必须实现Appender接口。大多数Appender将扩展AbstractAppender，它增加了Lifecycle和Filterable支持。Lifecycle允许组件在配置完成后完成初始化并在关闭期间执行清理。Filterable允许组件在事件处理过程中评估附加的filter。

Appender通常只负责将事件数据写入目的地。在大多数情况下，他们将格式化事件的责任委托给layout。一些appender会包装其他appender，以便修改LogEvent，或处理一个Appender中的故障，或者根据高级过滤标准将事件路由到下级Appender，或者提供不直接格式化事件以供查看类似的功能。

Appender总是有一个名字，以便它们可以从Logger中引用。

在下表中，Type列对应于预期的Java类型。对于非JDK类，除非另有说明，否则这些类通常应该位于Log4j Core中。

### 7.1 异步Appender

AsyncAppender接受对其他Appender的引用，并使LogEvent在单独的Thread上写入这些Appender。请注意，写入这些Appender时的exception将从应用程序中隐藏。AsyncAppender应该在它引用的appender之后进行配置，以允许其正常关闭。

默认情况下，AsyncAppender使用的`java.util.concurrent.ArrayBlockingQueue`不需要任何外部库。请注意，在使用此appender时，多线程应用程序应该小心：阻塞队列容易受到抢占锁的影响，测试表明当有更多线程同时记录时，性能可能会变差。可考虑使用无锁异步记录器以获得最佳性能。AsyncAppender参数如下：

- `AppenderRef，String`：要异步调用的Appender的名称，可配置多个AppenderRef元素。
- `blocking，boolean`：如果为true，则appender将等待队列中有空闲插槽。如果为false，则如果队列已满，则将事件写入错误appender。默认值是true。
- `shutdownTimeout，integer`：Appender在关闭时，应等待队列中的未处理日志事件刷新的时间。默认值为0意味着永远等待。
- `bufferSize，integer`：指定可以排队的最大事件数，默认值为1024。请注意，使用disruptor风格的BlockingQueue时，此缓冲区大小必须是2的幂。当应用程序的log的速度快于底层appender可以跟上填满队列的速度，行为将由AsyncQueueFullPolicy确定。
- `errorRef，String`：如果没有任何appender可以被调用，或者由于appender中的错误，或者队列已满而被调用的Appender的名称。如果未指定，则错误将被忽略。
- `filter，Filter`：用来确定该Appender是否应该处理事件。使用CompositeFilter时可以指定多个Filter。
- `name，String`：Appender的名称。
- `ignoreExceptions，boolean`：缺省值为true，导致在追加事件时遇到异常时，这些事件将在内部记录并被忽略。当设置为false时，异常将被传播给调用者。将此Appender包装在FailoverAppender中时，必须将其设置为false。
- `includeLocation，boolean`：提取位置是一项昂贵的操作（它可以使日志记录慢5到20倍）。为了提高性能，将日志事件添加到队列时，默认情况下不包括位置。你可以通过设置`includeLocation = true`来改变它。
- `BlockingQueueFactory，BlockingQueueFactory`：该元素重写要使用的BlockingQueue的类型。
