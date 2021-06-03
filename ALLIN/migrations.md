---
sort: 1
title: 🎉平滑迁移详细流程
tag: RocketMQ 平滑 迁移
---

[PDF下载]({{ site.baseurl }}/assets/pdf/RocketMQ迁移详细流程.pdf)

# 前言

​		由于早期自建RocketMQ服务部署在经典网络，应运维大哥要求，需要迁入VPC环境，所以产出这份迁移流程文档，并在一周时间内完成线上平滑迁移。本次没有升级版本，仍然以`4.1.0-incubating`老版本迁移部署。有升级需要可参考[丁威](https://blog.csdn.net/prestigeding)老师的[《线上环境大规模RocketMQ集群不停机优雅升级实践》](https://blog.csdn.net/prestigeding/article/details/115803949)。罗列几条注意事项：

- 避免使用root用户部署，方便后期非root用户查日志等（最好运维人员部署）。

- 执行`mqbroker`命令，要清楚目的，尤其是指定新/老`NameSrv`地址。

- "合理"分配`Broker`堆大小，默认启动脚本是`8g`。

- 不太清楚RocketMQ架构设计时，建议先看下[中文文档](https://wesleyone.github.io/rocketmqCake/design/architecture.html)。

- 深入学习推荐看丁威老师的`《RocketMQ技术内幕》`，搜公众号`中间件兴趣圈`。

​		本文亮点章节主要是[1.6 配置新Broker的Topic](#1.6 配置新Broker的Topic)和[3.2.1 查找业务应用](#3.2.1 查找业务应用)，解决了Topic数量较多时的"快速"创建，以及生产者消费者所在应用的定位。





# 流程简图

[drawio文件下载]({{ site.baseurl }}/assets/drawio/rocketmq迁移草案20210526.drawio)

**迁移前状态**

![迁移前状态]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移前的架构.jpg)


**搭建新RocketMQ服务**

![搭建新RocketMQ服务]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移1.jpg)



**关闭老Broker写入权限**

![关闭老Broker写入权限]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移2.jpg)



**更换使用新的NameSrv地址**

![更换使用新的NameSrv地址]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移3.jpg)



**新Broker移除老NameSrv地址**

![新Broker移除老NameSrv地址]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移4.jpg)



**迁移后状态**

![下线老RocketMQ服务]({{ site.baseurl }}/image/rocketmq迁移草案20210526-迁移完成.jpg)



---



# 一、搭建新RocketMQ服务

## 1.1. 源码打包

### 1.1.1 下载源码

> 使用成品包可忽略本步骤

源码地址 [rocketmq-all-4.1.0-incubating](https://github.com/apache/rocketmq/archive/refs/tags/rocketmq-all-4.1.0-incubating.zip)

### 1.1.2 打包

> 使用成品包可忽略本步骤

**1.1.2.1 在项目根目录下执行**

```
mvn -Prelease-all -DskipTests clean install -U
```

**1.1.2.2 获取产出可执行文件**

`distribution/target/apache-rocketmq.zip`



## 1.2 环境准备及配置

### 1.2.1 上传文件

将`apache-rocketmq.zip`文件传到`/home/mquser/app`路径下，并解压`unzip apache-rocketmq.zip `。关注的文件路径如下：

```shell
/home/mquser/app/apache-rocketmq
├── bin
│   ├── mqadmin
│   ├── mqbroker
│   ├── mqnamesrv
│   ├── mqshutdown
│   └── ......
├── conf
│   ├── broker.conf
│   └── ......
├── lib
│   ├── rocketmq-broker-4.1.0-incubating.jar
│   ├── rocketmq-namesrv-4.1.0-incubating.jar
│   └── ......
└── ......
```



### 1.2.2 配置环境变量

**1.2.2.1 安装JDK**

> 未安装JDK时需要

```
yum install java-1.8.0-openjdk.x86_64
```

**1.2.2.2 编辑环境变量配置**

```
vim /etc/profile
```

**添加JAVA_HOME**

```
export JAVA_HOME=/usr/lib/jvm/jre-1.8.0
export CLASSPATH=.:/jre/lib/rt.jar:/lib/dt.jar:/lib/tools.jar
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin
```

**添加ROCKETMQ_HOME**

```
export ROCKETMQ_HOME=/home/mquser/app/apache-rocketmq
```

**1.2.2.3 立即生效配置**

```
source /etc/profile
```



## 1.3 搭建NameSrv

> 部署两台NameSrv，流程完全相同

### 1.3.1 准备工作

登录`NameSrv`用的服务器，完成[1.2 环境准备及配置](#1.2 环境准备及配置)

### 1.3.2 运行NameSrv启动脚本

**1.3.2.1. 进入脚本目录**

```
cd /home/mquser/app/apache-rocketmq/bin
```

**1.3.2.2. 启动NameSrv**

```
nohup sh $ROCKETMQ_HOME/bin/mqnamesrv > $ROCKETMQ_HOME/namesrv_nohup.out &
```

**1.3.2.3. 验证启动进程**

```
ps ax | grep -i 'org.apache.rocketmq.namesrv.NamesrvStartup' |grep java | grep -v grep
```

**1.3.2.4. 验证日志**

```
tail -200f ~/logs/rocketmqlogs/namesrv.log
```

**1.3.2.5. 配置域名**（可选）

> 内网环境用域名



## 1.4 搭建Broker

> 按照双主无从部署，仅配置文件不同

### 1.4.1 准备工作

登录`Broker`用的服务器，完成[1.2 环境准备及配置](#1.2 环境准备及配置)

### 1.4.2 修改配置

**1.4.2.1. 进入配置目录**

```
cd /home/mquser/app/apache-rocketmq/conf
```

**1.4.2.2. 编辑配置**

**1.4.2.2.1 创建配置文件**

- Broker-C

```
vim broker_c.conf
```

- Broker-D

```
vim broker_d.conf
```

**1.4.2.2.2 添加内容并保存**

>  `namesrvAddr`需要获取实际值填充
>  具体配置可以参考老Broker配置
>  属性含义查看[Broker配置](https://wesleyone.github.io/rocketmqCake/best_practice/best_practice.html#33-broker-%E9%85%8D%E7%BD%AE)

```
brokerClusterName=DefaultClusterbrokerId=0deleteWhen=04fileReservedTime=168brokerRole=ASYNC_MASTERflushDiskType=ASYNC_FLUSH
```

NameSrv地址包括老的NameSrv地址和新的NameSrv地址:

```
namesrvAddr=NEW_NAMESRV_ADDR_1:9876;NEW_NAMESRV_ADDR_2.ltd:9876;OLD_NAMESRV_ADDR:9876
```

- Broker-C追加

```
brokerName=broker-c
```

- Broker-D追加

```
brokerName=broker-d
```



### 1.4.3 运行Broker启动脚本

**1.4.3.1. 进入脚本目录**

```
cd /home/mquser/app/apache-rocketmq/bin
```

**1.4.3.2. 启动Broker**

- Broker-C

```
nohup sh $ROCKETMQ_HOME/bin/mqbroker -c $ROCKETMQ_HOME/conf/broker_c.conf > $ROCKETMQ_HOME/broker_nohup.out &
```

- Broker-D

```
nohup sh $ROCKETMQ_HOME/bin/mqbroker -c $ROCKETMQ_HOME/conf/broker_d.conf > $ROCKETMQ_HOME/broker_nohup.out &
```

**1.4.3.3. 验证启动进程**

```
ps ax | grep -i 'org.apache.rocketmq.broker.BrokerStartup' |grep java | grep -v grep
```

**1.4.3.4. 验证日志**

```
tail -f ~/logs/rocketmqlogs/broker.log
```



## 1.5 搭建新Console控制台

### 1.5.1 下载源码

> 使用成品忽略本步骤

下载 源码[rocketmq-console-1.0.0](https://github.com/apache/rocketmq-externals/archive/refs/tags/rocketmq-console-1.0.0.zip)并解压，进入解压目录。

### 1.5.2 打包

> 使用成品忽略本步骤

**1.5.2.1 进入`rocketmq-console`项目**

```
cd rocketmq-console
```

**1.5.2.2 打包**

```
mvn -Prelease-all -DskipTests clean install -U
```

**1.5.2.3 获取成品JAR包**

`target/rocketmq-console-ng-1.0.0.jar`

### 1.5.3 获取文件并上传

上传成品包`rocketmq-console-ng-1.0.0.jar`到新服务器的`/home/mquser/app/apache-rocketmq`目录下。

> 可以部署在一台新NameSrv所在的服务器上

**1.5.3.1 进入工作目录**

```
cd /home/mquser/app/apache-rocketmq
```

**1.5.3.2 启动控制台**

> `NEW_NAMESRV_ADDR`修改为**新的NameSrv**的IP地址。

```
nohup java -Xmx256m -Xms256m -jar rocketmq-console-ng-1.0.0.jar --server.port=9800 --rocketmq.config.namesrvAddr=NEW_NAMESRV_ADDR:9876 > console.out &
```



## 1.6 配置新Broker的Topic

### 1.6.1 查看老Broker的业务Topic

> `OLD_NAMESRV_ADDR`一定要用**老NameSrv**的IP地址替换。

```
sh $ROCKETMQ_HOME/bin/mqadmin topicList -n 'OLD_NAMESRV_ADDR:9876' | grep -v '%RETRY%' | grep -v '%DLQ%' | grep -v 'BenchmarkTest' | grep -v 'TBW102' | grep -v 'rmq_sys_' | grep -v 'OFFSET_MOVED_EVENT' | grep -v 'DefaultCluster' | grep -v 'SELF_TEST_TOPIC' | grep -v 'broker-'
```

### 1.6.2 为所有新Broker创建已有Topic

> 将上一步查出来的`Topic`，拼接到如下创建`Topic`命令；
>
> `NEW_NAMESRV_ADDR`一定要配置为**新的NameSrv**的IP地址。
>
> `YOUR_TOPIC`修改为具体的Topic名称。
>
> 因为新NameSrv下只有新的Broker，所以只会给所有新的Broker创建相同配置的Topic，不影响老Broker。
>
> 命令参数含义见[参考文档-Topic相关](https://wesleyone.github.io/rocketmqCake/operation/operation.html#21-topic%E7%9B%B8%E5%85%B3)

```
sh $ROCKETMQ_HOME/bin/mqadmin updateTopic -n NEW_NAMESRV_ADDR:9876 -c DefaultCluster -p 6 -r 4 -w 4 -t YOUR_TOPIC
```

### 1.6.3 确认所有Topic在新Broker中消费

> `OLD_NAMESRV_ADDR`要用**老NameSrv**的IP地址替换。
>
> `YOUR_TOPIC`修改为具体的Topic名称。
>
> 命令参数含义见[参考文档-Topic相关](https://wesleyone.github.io/rocketmqCake/operation/operation.html#21-topic%E7%9B%B8%E5%85%B3)

**1.6.3.1 查看Topic下所有消息队列**

```
sh $ROCKETMQ_HOME/bin/mqadmin topicStatus -n 'OLD_NAMESRV_ADDR:9876' -t YOUR_TOPIC
```

**1.6.3.2 查看Topic路由**

```
sh $ROCKETMQ_HOME/bin/mqadmin topicRoute -n 'OLD_NAMESRV_ADDR:9876' -t YOUR_TOPIC
```

**1.6.3.3 查看集群下各Broker吞吐信息**

> `-i`刷新间隔秒数
>
> 命令参数含义见[参考文档-集群相关](https://wesleyone.github.io/rocketmqCake/operation/operation.html#22-%E9%9B%86%E7%BE%A4%E7%9B%B8%E5%85%B3)

- 查看各Broker的消息出入TPS

```
sh $ROCKETMQ_HOME/bin/mqadmin clusterList -n 'OLD_NAMESRV_ADDR:9876' -i 5
```

- 查看各Broker的消息出入数量

```
sh $ROCKETMQ_HOME/bin/mqadmin clusterList -n 'OLD_NAMESRV_ADDR:9876' -i 5 -m
```



# 二、关闭老Broker写入权限

## 2.1 处理时机

确认所有Topic已经在新的Broker中生成，保险起见再确保下消息有出入（以防网络不通等问题）。



## 2.2 关闭Broker写入

> `OLD_BROKER_IP`为`老Broker`的IP地址
>
> 命令参数含义见[参考文档-Broker相关](https://wesleyone.github.io/rocketmqCake/operation/operation.html#23-broker%E7%9B%B8%E5%85%B3)

**2.2.1 登录老Broker服务器**

**2.2.2 关闭`老Broker`写权限**

> ⚠️注意：执行这个命令会更新配置文件，把一些系统默认配置也写到配置文件中。

```
sh $ROCKETMQ_HOME/bin/mqadmin updateBrokerConfig -b 'OLD_BROKER_IP:10911' -k brokerPermission -v 4
```

**2.3.2 验证`老Broker`写权限关闭成功**

```
sh $ROCKETMQ_HOME/bin/mqadmin getBrokerConfig -b 'OLD_BROKER_IP:10911' | grep brokerPermission
```

结果应该为`4`。

**2.3.3 业务异常，恢复Broker权限**

```
sh $ROCKETMQ_HOME/bin/mqadmin updateBrokerConfig -b 'OLD_BROKER_IP:10911' -k brokerPermission -v 6
```

**查看恢复写入操作正确性**

```
sh $ROCKETMQ_HOME/bin/mqadmin getBrokerConfig -b 'OLD_BROKER_IP:10911' | grep brokerPermission
```

结果应该为`6`。

**2.3.4 观察`老Broker`消息出入状况**

见[1.6.3.3 查看集群下各Broker吞吐信息](#1.6.3 确认所有Topic在新Broker中消费)



# 三、更换使用新的NameSrv地址

## 3.1 处理时机

必须确认老Broker无可消费消息。



## 3.2 业务应用

### 3.2.1 查找业务应用

> 没有找到比较好的mqadmin查询命令，人工排查
>
> 后来发现[#2940](https://github.com/apache/rocketmq/pull/2940)丁威老师提的PR，方便查询生产者。
>
> 以下是运维大哥教的方法。

登录老NameSrv，查看访问NameSrv端口的网络IP，间接查找客户端应用（包括发送者和消费者）。

```
ss -natp |grep 9876
```

例如返回如下：

```
省略 [::ffff:198.0.0.0]:9876    [::ffff:198.0.0.1]:42430  省略
```

然后登录右边IP`198.0.0.1`的服务器，执行命令查询`42430`端口占用的进程ID：

```
ss -natp |grep 42430
```

例如返回如下：

```
省略 users:(("java",pid=12740,fd=61))
```

其中`pid=12740`就是相应应用的进程ID，可执行以下命令查看进程信息：

```
ps -ef|grep 12740
```

返回结果就能定位到具体应用了。



### 3.2.2 更换成新NameSrv地址

更换应用中的`NameSrv`地址为全部`新的NameSrv`地址。

### 3.2.3 发布业务项目

先发生产者，再发消费者。



# 四、新Broker移除老NameSrv地址

## 4.1 修改新Broker配置

```
cd /home/mquser/app/apache-rocketmq/conf
```

- Broker-C

```
vim broker_c.conf
```

- Broker-D

```
vim broker_d.conf
```

修改配置为：

````
namesrvAddr=NEW_NAMESRV_ADDR_1:9876;NEW_NAMESRV_ADDR_2.ltd:9876
````



## 4.2 逐台重启新Broker

**关闭Broker**

```
sh $ROCKETMQ_HOME/bin/mqshutdown broker
```

**启动Broker及验证**

见[1.4.3 运行Broker启动脚本](#1.4.3 运行Broker启动脚本)



# 五、下线老RocketMQ服务

## 5.1 关闭老Broker

**登录老Broker服务器**

**关闭老Broker**

```
sh $ROCKETMQ_HOME/bin/mqshutdown broker
```

**验证**

```
ps ax | grep -i 'org.apache.rocketmq.broker.BrokerStartup' |grep java | grep -v grep
```



## 5.2 关闭老NameSrv

**登录老NameSrv服务器**

**关闭老Broker**

```
sh $ROCKETMQ_HOME/bin/mqshutdown namesrv
```

**验证**

```
ps ax | grep -i 'org.apache.rocketmq.namesrv.NamesrvStartup' |grep java | grep -v grep
```



## 5.3 关闭老控制台

**将老控制台域名转移到新控制台服务上**

**登录老控制台服务器**

**查找进程**

```
ps ax | grep -i 'rocketmq-console-ng' |grep java | grep -v grep | awk '{print $1}'
```

**杀掉进程**

```
kill 进程号
```

