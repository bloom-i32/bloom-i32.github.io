---
title: 各种python库的小例子
published: 2024-04-29
description: '主要整理了python连接其他组件所需要的库的基本使用'
tags: [python, MQ]
category: 'python'
draft: false 
---

- [MQ](#mq)
  - [aio\_pika](#aio_pika)
- [DB](#db)


# MQ 
## aio_pika
> `aio_pika`是python用来连接rabbitMQ的异步客户端

下面是一个使用`aio_pika`连接rabbitmq收发消息的小例子。  

`receive_logs_topic.py` 用来接收匹配topic的消息
```python
import asyncio
import sys

from aio_pika import ExchangeType, connect
from aio_pika.abc import AbstractIncomingMessage


async def main() -> None:
    # 创建连接
    connection = await connect('amqp://guest:guest@124.71.84.245/')
    async with connection:
        # 创建管道
        channel = await connection.channel()
        # 一次处理prefetch_count条消息
        await channel.set_qos(prefetch_count=1)
        
        # 声明交换机
        topic_logs_exchange = await channel.declare_exchange(
            'topic_logs', # 交换机名称
            ExchangeType.TOPIC # 交换机类型
        )

        # 声明一个队列，exclusive=True表示为临时队列
        queue = await channel.declare_queue(exclusive=True)

        # 路由键，让消费者绑定匹配的队列
        binding_keys  = sys.argv[1:]
        if not binding_keys :
            sys.stderr.write(f'Usage: {sys.argv[0]} [binding_key]...\n')
            sys.exit(1)

        for binding_key in binding_keys:
            # 消费者绑定交换机上所有匹配路由键的队列
            await queue.bind(topic_logs_exchange, routing_key=binding_key)

        # 处理消息
        async with queue.iterator() as queue_iter:
            message: AbstractIncomingMessage
            async for message in queue_iter:
                async with message.process():
                    print(f" [x] {message.routing_key!r}:{message.body!r}")

        print(" [*] Waiting for messages. To exit press CTRL+C")
        await asyncio.Future()


if __name__ == '__main__':
    asyncio.run(main())
```



`emit_log_topic.py` 用来发送日志到指定的topic
```python
import asyncio
import sys

from aio_pika import DeliveryMode, ExchangeType, Message, connect


async def main() -> None:
    # 创建连接
    connection = await connect('amqp://guest:guest@124.71.84.245/')
    async with connection:
        # 创建管道
        channel = await connection.channel()
        
        # 声明交换机
        topic_logs_exchange = await channel.declare_exchange(
            'topic_logs', # 交换机名称
            ExchangeType.TOPIC # 交换机类型
        )

        # 消息发送到哪个topic
        routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'

        # 创建要发送的消息
        message_body = b" ".join(arg.encode() for arg in sys.argv[2:]) or b"Hello World!"
        message = Message(
            message_body, # 消息体
            delivery_mode=DeliveryMode.PERSISTENT # 消息持久化
        )

        
        await topic_logs_exchange.publish(
            message, # 发送的消息
            routing_key=routing_key # 发送到哪个topic上
        )

        print(f" [x] Sent {message.body!r}")



if __name__ == '__main__':
    asyncio.run(main())
```

# DB
