---
tags:
- Kibana
- Elasticsearch
- 大数据
key: 20180423
---
# Kibana 与 Elasticsearch 通讯分析(2)
上一篇中我们将通讯分为了两类。在这一篇我们具体分析一下两类通讯的区别。
<!--more-->

## 影响所有通讯的Kibana配置
如下配置将会影响所有两种通讯：
```yaml
#elasticsearch.ssl.certificate: /path/to/your/client.crt
#elasticsearch.ssl.key: /path/to/your/client.key
#elasticsearch.ssl.certificateAuthorities: [ "/path/to/your/CA.pem" ]
#elasticsearch.ssl.verificationMode: full
#elasticsearch.pingTimeout: 1500
#elasticsearch.requestTimeout: 30000
#elasticsearch.customHeaders: {}
```

主要影响的是跟服务器HTTP/HTTPs连接相关的配置，例如使用什么SSL certificate， 是否验证服务器SSL certificate，连接超时等等。

`elasticsearch.customHeaders` 这个配置稍微有些特殊，这个配置可以设置成`key: value` pair，设置成功后，所有发生在Kibana与Elasticsearch之间的通讯都会含有该HTTP header,例如：
```
elasticsearch.customHeaders: { "userName":"admin"}
```
这样在所有的通讯中，都会加上这个HTTP header。即使你配置了`elasticsearch.requestHeadersWhitelist`，这个值也不会被替换。

## 服务器通讯

服务器通讯是由Kibana服务器发起的通讯，一般不含具体用户数据的查询。大多数都是对Elasticsearch服务器基本状态的HEAD，GET操作。
在Kibana的配置文件中，如下配置将会影响到改通讯所包含的Header内容
```yaml
elasticsearch.username: "user"
elasticsearch.password: "pass"
```
在Kibana的文档是这样介绍这两个配置的，如果你的Elasticsearch被Basic authentication（基本认证）所保护的。这些配置中的用户名与密码是Kibana服务器启动时用来进行对Kibana索引进行维护的。你的Kibana用户对Elasticsearch服务器的访问（通过Kibana服务器代理）依然需要通过鉴权。
这里所说的启动时候对kibana索引进行维护的操作就是我所说的服务器通讯。而通过Kibana服务器代理的，Kibana用户对Elasticsearch服务器进行的访问就是我所说的客户端通讯。

## 客户端通讯

客户端通讯是由浏览器（Kibana用户）发起的，通过Kibana服务器代理发送到Elasticsearch服务器的通讯。在Kibana的配置文件中，如下配置将会影响这些访问的HTTP header
```yaml
elasticsearch.requestHeadersWhitelist: [ authorization ]
```
当浏览器(Kibana用户)向Kibana发送请求时会包含很多Http header。例如在使用了基本认证服务时。用户为了通过鉴权，会在HTTP头中包含authorization头，如果你没有在该配置中打开该请求头的转发，那么Kibana在发往Elasticsearch的请求中将会抛弃从用户请求中发来的鉴权头。鉴权将会失败。

所以如果使用Nginx之类的反向代理对Elasticsearch进行了保护，那么在kibana的配置中，首先要设置一个通用用户，该用户应该可以对`.kibana`索引进行操作，配置到`elasticsearch.username` 与 `elasticsearch.password`。为了客户可以访问，还需要配置 `elasticsearch.requestHeadersWhitelist: [ authorization ]` 来将客户的鉴权信息由Kibana代理转发到elasticsearch的服务器。当然在实践中，如果你使用了Basic Authentication对Elasticsearch进行了保护，那么建议也同样用相同配置对Kibana进行保护，这样在所有从Kibana到Elasticsearch的请求中都能包含authorization信息，确保所有功能都能正常使用。

在Kibana 6.2.x版本的实验中。如果没有使用想用的配置对Kibana进行保护，即使设置了`elasticsearch.requestHeadersWhitelist: [ authorization ]`，在进行index pattern的操作中依然会有一些操作失败。

To be continue...
