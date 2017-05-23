## 初识RabbitMQ

在开发一个后台服务时，要调用其它的服务，而该服务对外进行了限频，因此，需要将收到的请求进行排队，然后再调用其它的服务，这里打算用消息队列解决这个问题。

### 1 应用场景

* 异步处理：收到请求后，将请求写入到队列，然后就返回，后端的应用程序再去处理队列的任务。
* 应用解耦：当一个系统要调用另一个系统时，可以使用消息队列对两个系统进行解耦。
* 流量削峰：在秒杀或者团购活动中，会由于流量过大，导致流量暴增，为了降低后端的这种突发负载，可以将前端的请求写入到消息队列，后台的也许系统再去读取消息队列中的请求。
* 日志处理：agent对日志进行采集，上报到消息队列，svr读取消息队列中的日志，进行相应的计算处理并保存起来。

综上所述，消息队列的主要作用是可以将一些并发的请求进行临时存储，后端可以依次处理这些请求，从而提高整个系统的稳定性。

### 2 RabbitMQ的环境搭建

* erlang：由于RabbitMQ是使用erlang编写的。
* rabbitmq_server
* pika库(python)

### 3 一个简单的例子

send.py
``` python
#!/usr/bin/env python  
import pika  

# 连接rabbitmq服务器
connection = pika.BlockingConnection(pika.ConnectionParameters(  
        host='localhost'))  
channel = connection.channel()  

# 创建一个名字为hello的队列  
channel.queue_declare(queue='hello')  

# 发送消息：消息头为hello，消息体为Hello World!  
channel.basic_publish(exchange='',  
                      routing_key='hello',  
                      body='Hello World!')  

# 断开连接
connection.close()  
```

recv.py
```python
#!/usr/bin/env python  
import pika  
  
connection = pika.BlockingConnection(pika.ConnectionParameters(  
        host='localhost'))  
channel = connection.channel()  

# 创建一个名字为hello的队列  
channel.queue_declare(queue='hello')  
  
print 'Waiting for messages.'  

# 消息处理的回调函数
def callback(ch, method, properties, body):  
    print "receive %s" % body 

# 绑定队列和消息处理函数  
channel.basic_consume(callback,  
                      queue='hello',  
                      no_ack=True)  

# 开始监听队列的消息  
channel.start_consuming()  
```

一切看起来都很自然，除了中间创建队列的部分：两个程序都调用queue_declare创建队列，肯定只有一个生效，那么为什么不只在一个里面调用这个呢？
由于两个程序的执行顺序是不确定的，有可能先执行发送的程序，也有可能先执行消费的程序，如果只在其中一个中调用，那么当另一个程序先执行时，就会由于没有这个队列而失败。况且，两个程序都调用了创建的函数，只要它们提供的队列的属性是一样的，就没有问题(如果创建队列时，提供的属性不一样，则会失败)。

### 4 exchange: direct、fanout、topic

是时候该讲讲RabbitMQ的基本模型了，在一般的理解中，RabbitMQ是一个消息队列服务器，我们在其中创建一个队列，然后将消息发送到这个队列，最后消费者从队列中读取消息进行处理。但是，RabbitMQ却不是采用这种模型，为了更灵活地对消息进行分发，RabbitMQ在队列前面再加了一层exchange，用于对消息进行路由。

生产者不会直接将消息发送到队列，而是将消息发送给某个exchange，然后exchange根据自身的类型和消息头决定该把这个消息放到哪个或者哪些队列中。这里说了，消息的路由根据的是exchange的类型和消息头，而exchange有三种类型：

* direct：找到绑定了该exchange，并且绑定的key是消息头的队列，将消息投递到这些队列中。
* fanout：将消息投递到绑定了该exchange的队列，无论绑定的key是什么。
* topic：找到绑定了该exchange的队列，并检查绑定的key是否与消息头"匹配"，如果匹配，则将消息投递到这些队列中。

根据以上的叙述，还需要理解三方面的内容：

* 创建exchange，并指定类型

```python
# 创建名为logs的exchange，并且类型是direct
channel.exchange_declare(exchange = 'logs',  
                         type = 'direct')
```

* 如何绑定exchange和队列

```python
# 将名为logs的exchange和名为hello的队列绑定，
# 并且绑定的key是hello，如上所述，
# 如果是direct的exchange，那么只有当消息头是hello的消息才会被路由到队列中
channel.queue_bind(exchange='logs',  
                   queue = 'hello',  
                   routing_key = 'hello')
```

* topic类型的exchange是如何判定匹配的

topic类型的exchange与queue绑定的key是一个由多个域组成的字符串，每个域之间用`.`分隔，每个域的含义由用户自由定义。例如，可以将绑定的key设置为`abc.def`，那么，当exchange收到主题为abc.def的消息时就会投递到对应的队列中。topic类型的exchange也提供两种通配符：

(1) `*`表示一个主题

(2) `#`表示0个或多个主题

例如，当绑定的key设置为`abc.*`，那么当消息的routing_key是abc.def时，就会投递到绑定的队列中，而消息的routing_key是abc.def.ghi或者abc时，则不会投递到绑定的队列中。当绑定的key设置为`abc.#`，那么只要消息routing_key的第一个单词是abc就会投递到绑定的队列。

### 5 持久化和消息确认

由于svr在工作时是将消息保存在内存中，为了保证队列以及队列中的消息不在服务器异常或者svr本身异常的情况下丢失，需要对队列和消息进行持久化。

* 队列的持久化

```python
channel.queue_declare(queue='hello', durable=True)
```

* 消息的持久化

```python
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = 2,
                      ))
```

持久化可以保证在服务器或者svr本身出现问题时可以不丢失数据。正常情况下当队列中的某个消息被消费者消耗后，就会从队列中删除，那么，如果队列在处理过程中出现了问题异常退出了，该消息没处理完就没有了。因此，需要有一种确认机制保证一条消息在正常消耗后，才从队列中删除。这就是RabbitMQ中的消息确认机制。

```python
def callback(ch, method, properties, body):
    print "receive: %s" % body

    # 处理完成后，发送确认
    ch.basic_ack(delivery_tag = method.delivery_tag)

# basic_consume有一个参数no_ack，默认是False，也就是默认开启确认机制
channel.basic_consume(callback,
                      queue='hello')
```

### 6 总结

RabbitMQ中主要的设计思想就是使用了exchange，exchange在前端负责接收消息，然后根据消息头和绑定关系确定投递到哪个队列中，并且提供了多种类型的exchange，可以有不同的分发策略。另外，为了保证队列和消息能够在异常情况下不丢失，提供了队列和消息的持久化；为了保证每个消息被正常处理，提供了消息确认机制。

### 参考文献

1 [消息队列应用场景](http://www.cnblogs.com/stopfalling/p/5375492.html)

2 [RabbitMQ从入门到精通](http://blog.csdn.net/column/details/rabbitmq.html)