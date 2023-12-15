---
title:  消息中间件-ActiveMQ
categories:
  - Web
tags:
  - 搜索引擎
date: 2017-10-22 09:34:21
---

## 1.1 JMS简介
>JMS（Java Messaging Service） 是 Java 平台上有关面向消息中间件的技术规范， 它便于消息系统中的Java应用程序进行消息交换,并且通过提供标准的产生、发送、 接收消息的接口简化企业应用的开发。

## 1.2 ActiveMQ简介
>ActiveMQ 是 Apache 出品， 最流行的， 能力强劲的开源消息总线。 ActiveMQ 是一个完全支持 JMS1.1 和 J2EE 1.4 规范的 JMS Provider 实现， 尽管 JMS 规范出台已经是很久的事情了，但是JMS在当今的J2EE应用中间仍然扮演着特殊的地位。

<!--more-->

对消息的传递有两种类型：

1.  一种是点对点(Queue)：消息的生产者和消费者都可以是多个，但是对同一条消息而言，消息的制造者只有一个，消费者也只有一个，生产者和消费者一一对应
2. 另一种是发布/订阅模式：即一个生产者产生并发送一个消息后，可以由多个消费者进行接收。

**JMS定义的五种消息正文格式**

* StreamMessage--Java原始值的数据流
* MapMessage--一套名称-值对
* TextMessage--一个字符串对象
* ObjectMessage--一个序列化的Java对象
* ByteMessage--一个字节的数据流

---
ActiveMQ安装：
下载并解压
```shell
tar -zxvf apache-activemq-5.12.0-bin.tar.gz
```
为apache-activemq-5.12.0赋权
```shell
chmod 777 apache-activemq-5.12.0
```
进入apache-activemq-5.12.0\bin，并且给activemq赋权
```shell
chmod 755 activemq 
```
启动
```shell
./activemq start
```
ActiveMQ的默认管理端口：8161，通信端口：61616

---
## ActiveMQ示例

#### 点对点模式
(1) 创建activemqDemo,引入依赖

```<dependency>
<groupId>org.apache.activemq</groupId>
<artifactId>activemq-all</artifactId>
<version>5.11.2</version>
</dependency>
```
(2) 创建QueueProducer(核心步骤)
```java
//1、创建连接工厂
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFaction("tcp://192.168.25.133:61616");
//2、获取连接
Connection connection = connectionFactory.createConnection();
//3、启动连接
connection.start();
/*
* 4、获取session
* 第 1 个参数 是否使用事务
* 第 2 个参数 消息的确认模式
*/
Session session = connection.createSession(false,Session.AUTO_ACKNOWLEDGE);
//5、创建队列对象,test-queue为消息队列的唯一标识
Queue queue = session.createQueue("test-queue");
//6、创建消息生产者对象
MessageProductor producer = session.createProducer(queue);
//7、创建文本消息
TextMessage textMessage = session.createTextMessage("ActiveMQ消息测试！");
//8、使用生产者发送消息
producer.send(textMessage);
//9、关闭资源
producer.close();
session.close();
connection.close();
```

(3)消息消费者
```java
//1、创建连接工厂
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.25.133:61616");
//2、获取连接
Connection connection = connectionFactory.createConnection();
//3、启动连接
connection.start();
//4、获取session
Session session = connection.createSession(false,Session.AUTO_ACKNOWLEDGE);
//5、创建队列对象
Queue queue = session.createQueue("test-queue");
//6、创建消息消费者对象
MessageConsumer consumer = session.createConsumer(queue);
//7、接收消息
consumer.setMessageListener(new MessageListener(){
    public void onMessage(Message message){
        //获取文本消息对象
        TextMessage textMessage = (TextMessage)message;
        try{
            String text = textMEssage.getText();//提取文本
            System.out.println(text);
        }catch(JMSException e){
           ·····
        }
    }
});
//8、等待键盘输入，以阻塞线程，是线程保持监听状态
System.in.read();
//9、关闭资源
consumer.close();
session.close();
connection.close();
```

#### 发布/订阅模式

(1)创建消息生产者

```java
//1、创建连接工厂
ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.25.133:61616");
//2、创建连接
Connection connection = connectionFactory.createConnection();
//3、启动连接
connection.start();
//4、创建会话
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
//5、创建一个订阅topic
Topic topic = session.createTopic("text-topic");
//6、创建消息生产者
MessageProducer producer = session.CreateProducer(topic);
//7、创建消息
TextMessage textMessage = session.createTextMEssage("这是一个topic消息测试");
//8、发送消息
producer.send(textMessage);
//9、关闭资源
producer.close();
session.close();
connection.close();
```
(2)创建消息消费者
```java
//1.创建连接工厂
ActiveMQConnectionFactory connectionFactory=new
ActiveMQConnectionFactory("tcp://192.168.25.133:61616");
//2.获取连接
Connection connection = connectionFactory.createConnection();
//3.启动连接
connection.start();
//4.获取 session
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
//5.创建主题对象
Topic topic = session.createTopic("test-topic");
//6.创建消息消费者
MessageConsumer consumer=session.createConsumer(topic);
//7、接收消息
consumer.setMessageListener(new MessageListener(){
    public void onMessage(Message message){
        TextMessage textMessage = (TextMessage)message;
        try{
            String text = textMessage.getText();
            System.out.println(text);
        }catch(){
            ·······
        }
    }
});
//8、等待键盘输入
System.in.read();
//9、关闭资源
consumer.close();
session.close();
connection.close();
```
