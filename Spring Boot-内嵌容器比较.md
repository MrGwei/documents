# Spring Boot-内嵌容器比较

[TOC]

Spring Boot内嵌容器支持Tomcat、Jetty、Undertow。为什么选择Undertow？
这里有一篇文章，时间 2017年1月26日发布的： 

## 参考： 

[Tomcat vs. Jetty vs. Undertow: Comparison of Spring Boot Embedded Servlet Containers][https://examples.javacodegeeks.com/enterprise-java/spring/tomcat-vs-jetty-vs-undertow-comparison-of-spring-boot-embedded-servlet-containers/ ]

文章打开比较慢，大致是一下几点： 

- spring boot 项目建立
- 如何修改 三种 内置的容器（默认是tomcat）
- 对三种配置进行测试（测试工具jmeter）：此处测试分为两种
  - 简单的String返回
  - 复杂的json结果返回
  - JVisualVM 查看请求过程中堆内存的变化。
- 分析结果：undertow 以微弱的优势胜出 undertow > tomcat > jetty
- 总结：例子并不能看出undertow 有很大的优势，但是可以看出的是，undertow 在适应新的趋势发展和尝试长连接（从响应头中可以看出）的使用。

## 回顾

​	SpringBoot 内置了三种servlet 容器供大家选择，默认的是tomcat，说明它还是大众最多的选择。另外，也可以看出另外两种也还是有自己独有的优势。
​	从另一方面来说，SpringBoot 提供的默认配置也不一定正确，对于版本的使用和兼容，不一定很全，还是需要根据压测之后才可以确定。 

## 附 undertow 配置参考

```properties
# 设置IO线程数, 它主要执行非阻塞的任务,它们会负责多个连接, 默认设置为可用的CPU 核数
# 不要设置过大，如果过大，启动项目会报错：打开文件数过多
server.undertow.io-threads=16

# 阻塞任务线程池, 当执行类似servlet请求阻塞IO操作, undertow会从这个线程池中取得线程
# 它的值设置取决于系统线程执行任务的阻塞系数，默认值是IO线程数*8
server.undertow.worker-threads=256

# 以下的配置会影响buffer,这些buffer会用于服务器连接的IO操作,有点类似netty的池化内存管理。默认为 JVM 可用的最大空间
# 每块buffer的空间大小,越小的空间被利用越充分，不要设置太大，以免影响其他应用，合适即可
server.undertow.buffer-size=1024

# 每个区分配的buffer数量 , 所以pool的大小是buffer-size * buffers-per-region
# 新版本已被废弃
server.undertow.buffers-per-region=1024

# 是否分配的直接内存(NIO直接分配的堆外内存)
server.undertow.direct-buffers=true
```

## undertow 和 jetty 对比

参考：[Undertow,Tomcat和Jetty服务器配置详解与性能测试](https://www.cnblogs.com/maybo/p/7784687.html)

- 都是基于NIO实现的高并发轻量级服务器。
- 对于服务器端，我们关注的重点不是连接超时时间、socket超时时间以及任务执行超时时间的配置，而是线程池配置，包括工作线程和IO线程的分配。
- Jetty 使用全局的线程配置，最小8，最大200.
- Undertow 用于IO线程数同CPU 核数，工作线程数=IO*8
- 在负载小的情况下，三款服务器都有很好的性能。
- 在负载逐渐增大的时候，jetty 因为是全局的线程池配置，会出现阻塞情况。undertow 和 tomcat 表现良好。
- tomcat 的IO线程不可通过SpringBoot 调整，而Undertow 可配置。