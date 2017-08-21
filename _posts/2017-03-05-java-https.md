---
layout: post
title:  "本文探讨https证书中没有配置好证书链导致的问题。"
date:   2017-08-08 15:12:12 +0800
categories: Java
author: 小米粒
keywords: java, final
description: 本文探讨https证书中没有配置好证书链导致的问题.
---

> 本文探讨https证书中没有配置好证书链导致的问题

我们网站要进行https改造，配置上购买的SSL证书后，浏览器访问正常，但是写了个java代码用httpcomponents调用https rest接口时报错：

```java:n
Exception in thread "main" javax.net.ssl.SSLHandshakeException: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

at sun.security.ssl.Alerts.getSSLException(Alerts.java:192)

at sun.security.ssl.SSLSocketImpl.fatal(SSLSocketImpl.java:1904)

at sun.security.ssl.Handshaker.fatalSE(Handshaker.java:279)

at sun.security.ssl.Handshaker.fatalSE(Handshaker.java:273)

at sun.security.ssl.ClientHandshaker.serverCertificate(ClientHandshaker.java:1446)

at sun.security.ssl.ClientHandshaker.processMessage(ClientHandshaker.java:209)

at sun.security.ssl.Handshaker.processLoop(Handshaker.java:901)

at sun.security.ssl.Handshaker.process_record(Handshaker.java:837)

at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:1023)

at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1332)

at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1359)

at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1343)

at org.apache.http.conn.ssl.SSLConnectionSocketFactory.createLayeredSocket(SSLConnectionSocketFactory.java:394)

at org.apache.http.conn.ssl.SSLConnectionSocketFactory.connectSocket(SSLConnectionSocketFactory.java:353)

at org.apache.http.impl.conn.DefaultHttpClientConnectionOperator.connect(DefaultHttpClientConnectionOperator.java:141)

at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.connect(PoolingHttpClientConnectionManager.java:353)

at org.apache.http.impl.execchain.MainClientExec.establishRoute(MainClientExec.java:380)

at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:236)

at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:184)

at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:88)

at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)

at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:184)

at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:82)

at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:107)

at com.duiba.activity.cmsweb.controller.DappConfigCtrl.main(DappConfigCtrl.java:1248)

at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)

at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)

at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)

at java.lang.reflect.Method.invoke(Method.java:606)

at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)

Caused by: sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

at sun.security.validator.PKIXValidator.doBuild(PKIXValidator.java:385)

at sun.security.validator.PKIXValidator.engineValidate(PKIXValidator.java:292)

at sun.security.validator.Validator.validate(Validator.java:260)

at sun.security.ssl.X509TrustManagerImpl.validate(X509TrustManagerImpl.java:326)

at sun.security.ssl.X509TrustManagerImpl.checkTrusted(X509TrustManagerImpl.java:231)

at sun.security.ssl.X509TrustManagerImpl.checkServerTrusted(X509TrustManagerImpl.java:126)

at sun.security.ssl.ClientHandshaker.serverCertificate(ClientHandshaker.java:1428)

... 25 more

Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target

at sun.security.provider.certpath.SunCertPathBuilder.engineBuild(SunCertPathBuilder.java:196)

at java.security.cert.CertPathBuilder.build(CertPathBuilder.java:268)

at sun.security.validator.PKIXValidator.doBuild(PKIXValidator.java:380)

... 31 more 
```
网上查了一堆资料，要么说要把网站证书放到某个目录下，要么要改代码，没有我想要的，因为不可能让其他开发者去做这些事情。

后来了解到证书链这回事，才知道如何解决这个问题。有关证书链可以读这里：
http://blog.sina.com.cn/s/blog_53ed87c10102vn8b.html

此问题产生的原因是因为我们运维配置证书时只使用了签发的证书，java客户端无法找到可信任的上级证书，所以报错。解决方法也很简单，把中级证书、根证书附加到签发证书后面就可以了，具体方法参考这里：
https://yq.aliyun.com/articles/26569
