![公众号]({{ site.baseurl }}/assets/images/qrcode.png){:.border.rounded}

# rocketmqCake🍰

> 🚀声明：本文档主要来源官方文档，仅用于学习交流。感谢原文大佬们🙏

## 学习资料及工具：

- [RocketMQ官网文档](http://rocketmq.apache.org/docs/quick-start/)
- [RocketMQ官方Github](https://github.com/apache/rocketmq)
- [rocketmq-externals扩展项目](https://github.com/apache/rocketmq-externals)


## Apache RocketMQ开发者指南
--------

##### 这个开发者指南是帮助您快速了解,并使用 Apache RocketMQ

### 1. 概念和特性

- [概念(Concept)](basic/concept.md)：介绍RocketMQ的基本概念模型。

- [特性(Features)](basic/features.md)：介绍RocketMQ实现的功能特性。 


### 2. 架构设计

- [架构(Architecture)](design/architecture.md)：介绍RocketMQ部署架构和技术架构。

- [设计(Design)](design/design.md)：介绍RocketMQ关键机制的设计原理，主要包括消息存储、通信机制、消息过滤、负载均衡、事务消息等。


### 3. 样例

- [样例(Example)](example/RocketMQ_Example.md) ：介绍RocketMQ的常见用法，包括基本样例、顺序消息样例、延时消息样例、批量消息样例、过滤消息样例、事务消息样例等。


### 4. 最佳实践
- [最佳实践（Best Practice）](best_practice/best_practice.md)：介绍RocketMQ的最佳实践，包括生产者、消费者、Broker以及NameServer的最佳实践，客户端的配置方式以及JVM和linux的最佳参数配置。
- [消息轨迹指南(Message Trace)](best_practice/msg_trace/user_guide.md)：介绍RocketMQ消息轨迹的使用方法。
- [权限管理(Auth Management)](best_practice/acl/user_guide.md)：介绍如何快速部署和使用支持权限控制特性的RocketMQ集群。

- [Dledger快速搭建(Quick Start)](best_practice/dledger/quick_start.md)：介绍Dledger的快速搭建方法。

- [集群部署(Cluster Deployment)](best_practice/dledger/deploy_guide.md)：介绍Dledger的集群部署方式。

### 5. 运维管理
- [集群部署(Operation)](operation/operation.md)：介绍单Master模式、多Master模式、多Master多slave模式等RocketMQ集群各种形式的部署方法以及运维工具mqadmin的使用方式。



### 6. API Reference（待补充）

- [DefaultMQProducer API Reference](client/java/API_Reference_DefaultMQProducer.md)