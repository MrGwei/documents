# [Spring Boot-实现Undertow服务器同时支持HTTP2、HTTPS](https://segmentfault.com/a/1190000013777395)

[TOC]

## 前言

如今，企业级应用程序的高性能安全加密的常见场景是同时支持HTTP和HTTPS两种协议，这篇文章考虑如何让Spring Boot应用程序同时支持HTTP和HTTPS两种协议。Spring Boot的web容器已经有容器可以支持HTTP2了，这个例子中选择了Undertow高性能服务器作为Spring Boot的web容器。 

## What-什么是HTTP2

HTTP2是HTTP协议自1999年HTTP1.1发布后的首个更新，主要基于SPDY协议。由互联网工程任务组（IETF）的 Hypertext Transfer Protocol Bis（httpbis）工作小组进行开发。该组织于2014年12月将HTTP/2标准提议递交至IESG进行讨论，于2015年2月17日被批准。HTTP2标准于2015年5月以RFC7540正式发表。 

## Why-为什么要用HTTP2

 HTTP2是第二代的HTTP协议，关于HTTP2的优点这里就不阐述了，可以参考下面链接文章了解：[http://ju.outofmemory.cn/entr...](http://ju.outofmemory.cn/entry/346601)。
下图是Akamai 公司建立的一个官方的演示，主要用来说明在性能上HTTP/1.1和HTTP/2在性能升的差别。同时请求 379 张图片，HTTP/1.1加载用时4.54s，HTTP/2加载用时1.47s，大家可以通过 https://http2.akamai.com/demo 来感受下HTTP2的提速。 

## What-什么是HTTPS

要说HTTPS我们得先说SSL(Secure Sockets Layer，安全套接层)，这是一种为网络通信提供安全及数据完整性的一种安全协议，SSL在网络传输层对网络连接进行加密。SSL协议可以分为两层：SSL记录协议（SSL Record Protocol），它建立在可靠的传输协议如TCP之上，为高层协议提供数据封装、压缩、加密等基本功能支持；SSL握手协议（SSL Handshake Protocol），它建立在SSL记录协议之上，用于在实际数据传输开始之前，通信双方进行身份认证、协商加密算法、交换加密密钥等。在Web开发中，我们是通过HTTPS来实现SSL的。HTTPS是以安全为目标的HTTP通道，简单来说就是HTTP的安全版，即在HTTP下加入SSL层，所以说HTTPS的安全基础是SSL，不过这里有一个地方需要小伙伴们注意，就是我们现在市场上使用的都是TLS协议(Transport Layer Security，它来源于SSL)，而不是SSL，由于SSL出现较早并且被各大浏览器支持因此成为了HTTPS的代名词。

| http | https |
| :--: | :--: |
| HTTP |  HTTP  |
| TCP | SSL or TSL |
| IP | TCP |
|  | IP |

## Why-为什么要用HTTPS

超文本传输协议HTTP协议被用于在Web浏览器和网站服务器之间传递信息。HTTP协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了Web浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此HTTP协议不适合传输一些敏感信息，比如信用卡号、密码等。
为了解决HTTP协议的这一缺陷，需要使用另一种协议：安全套接字层超文本传输协议HTTPS。为了数据传输的安全，HTTPS在HTTP的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。
HTTPS和HTTP的区别主要为以下四点：
一、https协议需要到ca申请证书，一般免费证书很少，需要交费。
二、http是超文本传输协议，信息是明文传输，https 则是具有安全性的ssl加密传输协议。
三、http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。
四、http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全。

## How-如何使用HTTPS和HTTP

如果你使用Spring Boot，并且想在内嵌服务器中添加HTTPS，需要如下步骤：

- 要有一个SSL证书，买的或者自己生成的。
- 在Spring Boot中启动HTTPS。
- 将HTTP重定向到HTTPS（可选）。

### 在Spring Boot中启动HTTPS和HTTP2

将证书复制到Spring Boot应用的resources目录下， 在application.yml中配置证书及端口，密码填写第3步中的密码 

```
 ################---Undertow服务器支持HTTPS服务---################
 server:
   port: 8443
   http:
     port: 8082
   http2:
     enabled: true
   ssl:
     key-store: classpath:httcrootca.p12
     key-store-password: httcrootca@httc.com.cn
     key-store-type: PKCS12
     key-alias: httc-https-integration
   undertow:
     worker-threads: 20
     buffer-size: 512
     io-threads: 2
```

此配置会使Undertow容器监听8443端口，那么只有在域名前添加 https://才能访问网站内容，添加http://则不行，所以需要让Undertow容器监听8080端口，并将8080端口的所有请求重定向到8443端口，即完成http到https的跳转。 

### **支持HTTP、HTTPS**

```java
package cn.com.httc.www.https.config;

import io.undertow.Undertow;
import io.undertow.UndertowOptions;
import io.undertow.servlet.api.SecurityConstraint;
import io.undertow.servlet.api.SecurityInfo;
import io.undertow.servlet.api.TransportGuaranteeType;
import io.undertow.servlet.api.WebResourceCollection;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.embedded.undertow.UndertowBuilderCustomizer;
import org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory;
import org.springframework.boot.web.servlet.server.ServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SSLConfiguration {

    @Value("${server.http.port}")
    private Integer httpPort;

    @Bean
    public ServletWebServerFactory undertowFactory() {
        UndertowServletWebServerFactory undertowFactory = new UndertowServletWebServerFactory();
        undertowFactory.addBuilderCustomizers((Undertow.Builder builder) -> {
            builder.addHttpListener(httpPort, "0.0.0.0");
        });
        return undertowFactory;
    }

}
```

### **将HTTP重定向到HTTPS（可选）**

```java
package cn.com.httc.www.https.config;

import io.undertow.Undertow;
import io.undertow.UndertowOptions;
import io.undertow.servlet.api.SecurityConstraint;
import io.undertow.servlet.api.SecurityInfo;
import io.undertow.servlet.api.TransportGuaranteeType;
import io.undertow.servlet.api.WebResourceCollection;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.web.embedded.undertow.UndertowBuilderCustomizer;
import org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory;
import org.springframework.boot.web.servlet.server.ServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SSLConfiguration {

    @Value("${server.https.port}")
    private Integer httpPort;

    @Value("${server.port}")
    private Integer httpsPort;

    /**
     * 采用Undertow作为服务器。
     * Undertow是一个用java编写的、灵活的、高性能的Web服务器，提供基于NIO的阻塞和非阻塞API，特点：
     * 非常轻量级，Undertow核心瓶子在1Mb以下。它在运行时也是轻量级的，有一个简单的嵌入式服务器使用少于4Mb的堆空间。
     * 支持HTTP升级，允许多个协议通过HTTP端口进行多路复用。
     * 提供对Web套接字的全面支持，包括JSR-356支持。
     * 提供对Servlet 3.1的支持，包括对嵌入式servlet的支持。还可以在同一部署中混合Servlet和本机Undertow非阻塞处理程序。
     * 可以嵌入在应用程序中或独立运行，只需几行代码。
     * 通过将处理程序链接在一起来配置Undertow服务器。它可以对各种功能进行配置，方便灵活。
     */
    @Bean
    public ServletWebServerFactory undertowFactory() {
        UndertowServletWebServerFactory undertowFactory = new UndertowServletWebServerFactory();
        undertowFactory.addBuilderCustomizers((Undertow.Builder builder) -> {
            builder.addHttpListener(httpPort, "0.0.0.0");
            // 开启HTTP2
            builder.setServerOption(UndertowOptions.ENABLE_HTTP2, true);
        });
        undertowFactory.addDeploymentInfoCustomizers(deploymentInfo -> {
            // 开启HTTP自动跳转至HTTPS
            deploymentInfo.addSecurityConstraint(new SecurityConstraint()
                    .addWebResourceCollection(new WebResourceCollection().addUrlPattern("/*"))
                    .setTransportGuaranteeType(TransportGuaranteeType.CONFIDENTIAL)
                    .setEmptyRoleSemantic(SecurityInfo.EmptyRoleSemantic.PERMIT))
                    .setConfidentialPortManager(exchange -> httpsPort);
        });
        return undertowFactory;
    }

}
```

### 验证HTTPS和HTTP2开启成功

重启服务，即完成了HTTP到HTTPS的升级，且能自动跳转HTTPS，让网站更安全。输入[http://localhost](http://localhost/):8082/后回车地址栏自动跳转至[https://localhost](https://localhost/):8443/

使用Chrome浏览器的console控制台，输入脚本后回车查看HTTP2是否开启，脚本如下： 

```javascript
(function(){
    // 保证这个方法只在支持loadTimes的chrome浏览器下执行
    if(window.chrome && typeof chrome.loadTimes === 'function') {
        var loadTimes = window.chrome.loadTimes();
        var spdy = loadTimes.wasFetchedViaSpdy;
        var info = loadTimes.npnNegotiatedProtocol || loadTimes.connectionInfo;
        // 就以 「h2」作为判断标识
        if(spdy && /^h2/i.test(info)) {
            return console.info('本站点使用了HTTP/2');
        }
    }
    console.warn('本站点没有使用HTTP/2');
})();
```

## **总结**

本文只是介绍了Undertow服务器的HTTPS支持，Spring Boot支持Jetty,Tomcat等服务器，不同的服务器实现可以查资料了解。