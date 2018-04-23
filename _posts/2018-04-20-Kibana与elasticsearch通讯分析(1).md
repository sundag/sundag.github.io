---
tags:
- Kibana
- Elasticsearch
- 大数据
key: 20180420
---
# Kibana 与 Elasticsearch 通讯分析(1)
最近正在开发一个大数据相关项目，使用Elasticsearch与Kibana给客户提供图形化面板。因为项目需求研究了不少Kibana与Elasticsearch之间的通讯。在这里做一点记录。
<!--more-->

## 版本

项目开始时使用的是5.2.x的版本，随着项目推进逐渐转移到了6.x版本上。在这两个版本之间，kibana与elasticsearch之间的通讯并没有什么大的设计改变。只有在读取Kibana索引相关内容的时候有部分API的调用链更新。在接下来的文档中，如果没有特殊声明，则是在介绍 6.x 版本的行为。

## 基本情况

* Kibana会使用REST API来访问Elasticsearch。
* 你只能在Kibana上配置单独一个Elasticsearch的URL。
* 所有Kibana到Elasticsearch的访问都会使用该地址。
* 所以如果在生产环境使用Kibana的话，建议是在kibana的同一个server上运行一个仅启动负载均衡的Elasticsearch节点，这样Kibana到Elasticsearch集群的访问可以通过该节点平均分配到各个Elastic节点。

## 通讯分类

Kibana与Elasticsearch服务器之间的通讯可以分为两类：
1. 客户通讯 - 由客户服务器发起，经由Kibana服务器到达Elasticsearch服务器的请求
2. 服务器通讯 - 由Kibana服务器发起，到达Elasticsearch服务器的请求

```mermaid
graph LR
A(浏览器) -->B(Kibana)
    B -->|服务器通讯| C(Elasticsearch)
    B -->|客户端通讯| C(Elasticsearch)
​```

To be continue...
