---
title: 用 RabbitMQ 的死信队列来做定时任务
date: 2018-10-05 18:26:50
tags: [RabbitMQ]
---

在开发中做定时任务是一个非常常见的业务场景，在代码层面 Node.js 可以用 setTimeout、setInerval 这种基础语法或用 [node-schedule](https://www.npmjs.com/package/node-schedule) 这些类似的库来达到部分目的，在第三方服务上可以用 Redis 的 Keyspace Notification 或 Linux 自身的 crontab 来做定时任务。RabbitMQ 作为一个消息中间件，使用其死信队列也可以达到做定时任务的目的。<!-- more -->

![](http://nnblog-storage.b0.upaiyun.com/img/rabbitmq-1.png)

本文以 Node.js 作为演示语言，操作 RabbitMQ 使用的是 [amqplib](https://www.npmjs.com/package/amqplib)。

### 死信队列
RabbitMQ 中有一种交换器叫 DLX，全称为 Dead-Letter-Exchange，可以称之为死信交换器。当消息在一个队列中变成死信（dead message）之后，它会被重新发送到另外一个交换器中，这个交换器就是 DLX，绑定在 DLX 上的队列就称之为死信队列。
消息变成死信一般是以下几种情况：

- 消息被拒绝，并且设置 requeue 参数为 false
- 消息过期
- 队列达到最大长度

DLX 也是一个正常的交换器，和一般的交换器没有区别，它能在任何队列上被指定，实际上就是设置某个队列的属性。当这个队列存在死信时，RabbitMQ 就会自动地将这个消息重新发布到设置的 DLX 上去，进而被路由到另一个队列，即死信队列。要为某个队列添加 DLX，需要在创建这个队列的时候设置其`deadLetterExchange` 和 `deadLetterRoutingKey` 参数，`deadLetterRoutingKey` 参数可选，表示为 DLX 指定的路由键，如果没有特殊指定，则使用原队列的路由键。

```javascript
const amqp = require('amqplib');

const myNormalEx = 'my_normal_exchange';
const myNormalQueue = 'my_normal_queue';
const myDeadLetterEx = 'my_dead_letter_exchange';
const myDeadLetterRoutingKey = 'my_dead_letter_routing_key';
let connection, channel;
amqp.connect('amqp://localhost')
  .then((conn) => {
    connection = conn;
    return conn.createChannel();
  })
  .then((ch) => {
    channel = ch;
    ch.assertExchange(myNormalEx, 'direct', { durable: false });
    return ch.assertQueue(myNormalQueue, {
      exclusive: false,
      deadLetterExchange: myDeadLetterEx,
      deadLetterRoutingKey: myDeadLetterRoutingKey,
    });
  })
  .then((ok) => {
    channel.bindQueue(ok.queue, myNormalEx);
    channel.sendToQueue(ok.queue, Buffer.from('hello'));
    setTimeout(function () { connection.close(); process.exit(0) }, 500);
  })
  .catch(console.error);

```

上面的代码先声明了一个交换器 `myNormalEx`， 然后声明了一个队列 `myNormalQueue`，在声明该队列的时候通过设置其 `deadLetterExchange` 参数，为其添加了一个 DLX。所以当队列 `myNormalQueue` 中有消息成为死信后就会被发布到 `myDeadLetterEx` 中去。

### 过期时间（TTL）

在 RabbbitMQ 中，可以对消息和队列设置过期时间。当通过队列属性设置过期时间时，队列中所有消息都有相同的过期时间。当对消息设置单独的过期时间时，每条消息的 TTL 可以不同。如果两种方法一起使用，则消息的 TTL 以两者之间较小的那个数值为准。消息在队列中的生存时间一旦超过设置的 TTL 值时，就会变成“死信”（Dead Message），消费者将无法再接收到该消息。

针对每条消息设置 TTL 是在发送消息的时候设置 `expiration` 参数，单位为毫秒。

```javascript
const amqp = require('amqplib');

const myNormalEx = 'my_normal_exchange';
const myNormalQueue = 'my_normal_queue';
const myDeadLetterEx = 'my_dead_letter_exchange';
const myDeadLetterRoutingKey = 'my_dead_letter_routing_key';
let connection, channel;
amqp.connect('amqp://localhost')
  .then((conn) => {
    connection = conn;
    return conn.createChannel();
  })
  .then((ch) => {
    channel = ch;
    ch.assertExchange(myNormalEx, 'direct', { durable: false });
    return ch.assertQueue(myNormalQueue, {
      exclusive: false,
      deadLetterExchange: myDeadLetterEx,
      deadLetterRoutingKey: myDeadLetterRoutingKey,
    });
})
  .then((ok) => {
    channel.bindQueue(ok.queue, myNormalEx);
    channel.sendToQueue(ok.queue, Buffer.from('hello'), { expiration: '4000'});
    setTimeout(function () { connection.close(); process.exit(0) }, 500);
  })
  .catch(console.error);

```

上面的代码在向队列发送消息的时候，通过传递 `{ expiration: '4000'}` 将这条消息的过期时间设为了4秒，对消息设置4秒钟过期，这条消息并不一定就会在4秒钟后被丢弃或进入死信，只有当这条消息到达队首即将被消费时才会判断其是否过期，若未过期就会被消费者消费，若已过期就会被删除或者成为死信。

### 定时任务

因为队列中的消息过期后会成为死信，而死信又会被发布到该消息所在的队列的 DLX 上去，**所以通过为消息设置过期时间，然后再消费该消息所在队列的 DLX 所绑定的队列，从而来达到定时处理一个任务的目的。** 简单的讲就是当有一个队列 queue1，其 DLX 为 deadEx1，deadEx1 绑定了一个队列 deadQueue1，当队列 queue1 中有一条消息因过期成为死信时，就会被发布到 deadEx1 中去，通过消费队列 deadQueue1 中的消息，也就相当于消费的是 queue1 中的因过期产生的死信消息。

![](http://nnblog-storage.b0.upaiyun.com/img/dead-letter.png)

消费死信队列的代码如下：

```javascript
const amqp = require('amqplib');

const myDeadLetterEx = 'my_dead_letter_exchange';
const myDeadLetterQueue = 'my_dead_letter_queue';
const myDeadLetterRoutingKey = 'my_dead_letter_routing_key';
let channel;
amqp.connect('amqp://localhost')
.then((conn) => {
  return conn.createChannel();
})
.then((ch) => {
  channel = ch;
  ch.assertExchange(myDeadLetterEx, 'direct', { durable: false });
  return ch.assertQueue(myDeadLetterQueue, { exclusive: false });
})
.then((ok) => {
  channel.bindQueue(ok.queue, myDeadLetterEx, myDeadLetterRoutingKey);
  channel.consume(ok.queue, (msg) => {
    console.log(" [x] %s: '%s'", msg.fields.routingKey, msg.content.toString());
  }, { noAck: true})
})
.catch(console.error);
```

这里需要注意的是，如果声明的 `myDeadLetterEx` 是 direct 类型，那么在为其绑定队列的时候一定要指定 BindingKey，即这里的 `myDeadLetterRoutingKey`，如果不指定 Bindingkey，则需要将 `myDeadLetterEx` 声明为 fanout 类型。