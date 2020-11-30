+++
author = "Vladimir Lukiyanov"
title = "Exploring Redis streams with Akka streams"
date = "2020-11-27"
lastmod = "2020-12-01"
description = "A short introduction to consuming and producing messages to Redis streams using Akka streams"
tags = [
    "akka", "redis", "streams", "scala", "exploration"
]
+++

In this post we will explore using Redis streams in Akka streams using the popular Java library [Lettuce](https://github.com/lettuce-io/lettuce-core); Redis is an in-memory database used in distributed systems, and Akka streams is a popular reactive streams implementation for Scala. The streams data structure in Redis implements a persistent log with support for consumer groups, this is a data structure designed to fan out messages to multiple consumers and there is a [good but lengthy introduction to Redis streams on the Redis website](https://redis.io/topics/streams-intro), but roughly speaking there are three classes of operation:

* **Adding messages to the stream** using the [XADD](https://redis.io/commands/xadd) operation; each message has a unique ID, which is usually generated automatically by the Redis server and returned to the caller.
* **Reading messages from the stream** using [XREAD](https://redis.io/commands/xread) and [XREADGROUP](https://redis.io/commands/xreadgroup) operations. There are various modes here, from just reading a given message by ID, to reading message using consumer groups.
* **Dealing with consumer groups, consumers, and acknowledging messages** using operations like [XACK](https://redis.io/commands/xack), [XGROUP](https://redis.io/commands/xgroup), and [XPENDING](https://redis.io/commands/xgroup).

In essence consumer groups allow fanning out messages to multiple consumers with guarantees around delivery, and the ability to inspect messages that were consumed but not acknowledged by consumers - properties that are essential for building reliable systems. To explore the Redis streams API we will create a simple app which will produce, consume and then acknowledge messages, a rough outline is the following:

{{<center>}}
{{<mermaid>}}
graph LR;
    A(Source of messages) --> B(XADD) --> C(Sink ignore)
{{< /mermaid >}}
{{</center>}}

{{<center>}}
{{<mermaid>}}
graph LR;
    E(Source of XREADGROUP) --> F(Sink XACK)
{{< /mermaid >}}
{{</center>}}

We will optimise this app for throughput, but note that the optimisations are unlikely to transfer equally to all environments.

Using Redis streams with Akka streams is possible using one of the Redis client libraries for Java in Scala, in particular [Redisson](https://github.com/redisson/redisson) and [Lettuce](https://github.com/lettuce-io/lettuce-core); in the following examples we use the simple async version of the latter. Note that Lettuce also has a [reactive interface](https://github.com/lettuce-io/lettuce-core/wiki/Reactive-API-(5.0)) using [Project Reactor](http://projectreactor.io/), this interface can be used within Akka Streams via the [reactive streams interop](https://doc.akka.io/docs/akka/current/stream/reactive-streams-interop.html) and might be suitable for a number of applications and environments.

# Setting up

To experiment with Redis streams we need a local server, using Docker some variation of `docker run --name redis-local -p 6379:6379 -d redis` will setup a local server on port `6379`; if you adjust the port number, using say `-p 6380:6379`, you will then need to change the connection string to match (controlled by the environment variable `REDIS_URL`)

The repo [vlukiyanov/akka-redis-streams-example](https://github.com/vlukiyanov/akka-redis-streams-example) contains full examples of a producer and a consumer.

# Producing messages

The producer publishes a `Map[String, String]` to a given stream using the [XADD](https://redis.io/commands/xadd) operation, this will then add the input to the stream with an ID based on the timestamp, which is then returned; the process can be modelled as a `Flow[Map[String, String], String, NotUsed]`. A simple implementation using Lettuce's `XACK` could be roughly the following:

```scala
object RedisStreamsFlow {
  def create(redis: RedisCommands[String, String],
             stream: String): Flow[Map[String, String], String, NotUsed] = {
    Flow
      .apply[Map[String, String]]
      .map(elem => redis.xadd(stream, elem.asJava))
  }
}
```

This is simple but not always performant, a more performant version can be implemented using pipelining. To do this compress the `Flow` using the `groupedWithin` method, then send the groups in `Seq[Map[String, String]]` chunks to create `Seq[String]` and finally apply `.mapConcat(identity)` to flatten the results into instances of `String`.

```scala
object RedisStreamsFlow {
  def create(redis: RedisAsyncCommands[String, String],
             stream: String): Flow[Map[String, String], String, NotUsed] = {
    redis.setAutoFlushCommands(false)
    Flow
      .apply[Map[String, String]]
      .groupedWithin(1000, 1.second)
      .mapAsync(4) { elems =>
        {
          val futures =
            elems.map(elem => redis.xadd(stream, elem.asJava).asScala)
          redis.flushCommands()
          Future.sequence(futures)
        }
      }
      .mapConcat(identity)
  }
}
```

In the above `redis.setAutoFlushCommands(false)` disables Lettuce's automatic flushing, as discussed in [Lettuce documentation](https://lettuce.io/core/release/reference/#_pipelining_and_command_flushing), and instead pipelines the commands manually. This periodically invokes writing to the transport layer using `redis.flushCommands()` after many commands have been added - increasing throughput. On my local system setting parallelism to 4 increased throughput to a maximum, given Redis is single threaded this is something to note, though may not be relevant in actual production scenarios.

The code for this is in [`RedisStreamsFlow.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/api/RedisStreamsFlow.scala) and [`ProducerExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/ProducerExample.scala).

# Consuming messages

There are many ways to consume messages using Redis streams, to make the most of the capabilities that stream offers we need to use consumer groups as discussed in the [introduction on the Redis website](https://redis.io/topics/streams-intro). Lettuce's [XREADGROUP](https://redis.io/commands/xreadgroup) implementation takes a consumer group, consumer name, an offset configuration for the reading the stream and returns a  list of `io.lettuce.core.StreamMessage`; a `StreamMessage` stores a string `id` and a `Map[String, String]` representing the `body` of the message consumed. We can model the consumer as a `Source[StreamMessage[String, String], Cancellable]`. An implementation is something like the following:

```scala
object RedisStreamsSource {
  def create(
      redis: RedisAsyncCommands[String, String],
      stream: String,
      group: String,
      consumer: String): Source[StreamMessage[String, String], Cancellable] = {
    Source
      .tick(0 millisecond, 100 millisecond, Done)
      .mapAsync(4) { _ =>
        redis
          .xreadgroup(
            Consumer.from(group, consumer),
            XReadArgs.Builder.count(100000),
            XReadArgs.StreamOffset.lastConsumed(stream)
          )
          .asScala
      }
      .mapConcat(_.asScala.toList)
  }
}
```

The parameters here need tuning, but this will poll Redis at some fixed interval, then flatten the lists of messages using `.mapConcat(_.asScala.toList)`. On my local system setting parallelism to 4 increased throughput to a maximum, given Redis is single threaded this is again something to note, though may not be relevant in actual production scenarios. The code for this is in [`RedisStreamsSource.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/api/RedisStreamsSource.scala) and [`ConsumerExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/ConsumerExample.scala).

# Acknowledging messages

When a consumer in a consumer group has successfully processed a message it should indicate this by sending an [XACK](https://redis.io/commands/xack); sending [XACK](https://redis.io/commands/xack) removes the message from the pending list. As a message is uniquely determined by a string id, this process can be modelled as `Sink[String, NotUsed]`, and in implementation is something like the following:

```scala
object RedisStreamsAckSink {
  def create(redis: RedisAsyncCommands[String, String],
             group: String,
             stream: String): Sink[String, NotUsed] = {
    Flow
      .apply[String]
      .mapAsync(4) { messageId =>
        redis.xack(stream, group, messageId).asScala
      }
      .to(Sink.ignore)
  }
}
```

# Threading it all together

We're now able to create a full example which simultaneously produces and consumes messages, afterwards acknowledging them. First the producer, this includes a snippet from https://stackoverflow.com/a/49279641 for rate measurement:

```scala
val redisStreamsFlow = RedisStreamsFlow.create(asyncCommands, "testStream")
val messageSource = Source.repeat(Map("key" -> "test")).limit(10_000_000)

messageSource
  .via(redisStreamsFlow)
  .conflateWithSeed(_ => 0) { case (acc, _) => acc + 1 }
  .zip(Source.tick(1.second, 1.second, NotUsed))
  .map(_._1)
  .toMat(Sink.foreach(i => println(s"$i elements/second producer")))(Keep.right)
  .run()
```

Then we can take the same stream and consume the produces message, acknowledging them as we go along:

```scala
val redisStreamsSource = RedisStreamsSource.create(asyncCommands,
  "testStream",
  "testGroup",
  "testConsumer")

val redisStreamAckSink = RedisStreamsAckSink.create(asyncCommands, "testGroup", "testStream")

redisStreamsSource
  .map(_.getId)
  .to(redisStreamAckSink)
  .run()
```

The code for this is in [`AckExample.scala`](https://github.com/vlukiyanov/akka-redis-streams-example/blob/main/src/main/scala/example/AckExample.scala), running this code we eventually see that all the message have been acknowledged by running [XPENDING](https://redis.io/commands/xpending):

```
127.0.0.1:6379> XPENDING testStream testGroup
1) (integer) 0
2) (nil)
3) (nil)
4) (nil)
```

This means that the code has successfully finished, and we have the example app; hopefully this can be used as a springboard to exploring other features in Redis stream using Akka streams, and possibly application for this in-memory log data structure.

