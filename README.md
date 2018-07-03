# RedisSMQ - Yet another simple Redis message queue

A simple high-performance Redis message queue for Node.js.

For more details about RedisSMQ design see [https://medium.com/@weyoss/building-a-simple-message-queue-using-redis-server-and-node-js-964eda240a2a](https://medium.com/@weyoss/building-a-simple-message-queue-using-redis-server-and-node-js-964eda240a2a)

## Features

 * **Persistent**: No messages are lost in case of a consumer failure.
 * **Atomic**: A message is delivered only once to one consumer (in FIFO order) so you would never fall into a situation
 where a message could be processed more than once.
 * **Fast**: 13K+ messages/second on a virtual machine of 4 CPU cores and 8GB RAM and running one consumer.
 * **Scalable**: A queue can be consumed by many concurrent consumers, running on the same or on different hosts.
 * **Message expiration**: A message will expire and not be consumed if it has been in the queue for longer than the 
 TTL (time-to-live).
 * **Message consume timeout**: The amount of time for a consumer to consume a message. If the timeout exceeds,
 message processing is cancelled and the message is re-queued to be consumed again.
 * **Delaying and scheduling message delivery**: From version 1.0.19 a persistent scheduler has been built into 
 RedisSMQ message queue. The scheduler accepts delaying messages, repeated messages delivery, period
 between repeats and CRON expressions.
 * **Highly optimized**: No promises, no async/await, small memory footprint, no memory leaks.
 * **Monitorable**: Statistics (input/processing/acks/unacks messages rates, online consumers, queues, etc.)
   are provided in real-time.
 * **Logging**: Supports JSON log format for troubleshooting and analytics purposes.
 * **Configurable**: Many options and features can be configured.  
 
## Installation

```text
npm install redis-smq --save
```

Considerations:

- Minimal Node.js version support is 7.0.0 (with --harmony flag), 7.6.0 (without --harmony flag). The latest stable 
  Node.js version is recommended.
- Minimal Redis server version is 2.6.12.


# Configuration

Before running a Producer or a Consumer instance, an object containing the configuration parameters can be supplied 
to the class constructor in order to configure the message queue.

A configuration object may look like:

```javascript
'use strict';

const path = require('path');

module.exports = {
    namespace: 'my_project_name',
    redis: {
        host: '127.0.0.1',
        port: 6379,
        connect_timeout: 3600000,
    },
    log: {
        enabled: 0,
        options: {
            level: 'trace',
            /*
            streams: [
                {
                    path: path.normalize(`${__dirname}/../logs/redis-smq.log`)
                },
            ],
            */
        },
    },
    monitor: {
        enabled: true,
        host: '127.0.0.1',
        port: 3000,
    },
};
```

**Parameters**

- `namespace` *(String): Optional.* The namespace for message queues. It can be composed only of letters (a-z), 
  numbers (0-9) and (-_) characters. Namespace can be for example configured per project. 

- `redis` *(Object): Optional.* Redis client parameters. 
  See https://github.com/NodeRedis/node_redis#options-object-properties for all valid parameters.

- `log` *(Object): Optional.* Logging parameters.

- `log.enabled` *(Integer/Boolean): Optional.* Enable/disable logging. By default logging is disabled.

- `log.options` *(Object): Optional.* All valid Bunyan configuration options are accepted. Please look at the 
  [Bunyan Repository](https://github.com/trentm/node-bunyan) for more details.

- `monitor` *(Object): Optional.* RedisSMQ monitor parameters.

- `monitor.enabled` *(Boolean/Integer): Optional.* Enable/Disable the monitor.

- `monitor.host` *(String): Optional.* IP address of the monitor server.

- `monitor.port` *(Integer): Optional.* Port of the monitor server.


## Usage

## Overview

RedisSMQ provides 3 classes: Message, Producer and Consumer in order to work with the message queue. 

### Message Class

Message class is the main component responsible for creating and handling messages. It encapsulates and provides all
the required methods needed to construct and deal with messages.

```javascript

const { Message } = require('redis-smq');

const message = new Message();

message
    .setBody({hello: 'world'})
    .setTTL(3600000)
    .setScheduledDelay(10)
    .setScheduledRepeat(6)
    .setScheduledPeriod(60)
    .setScheduledCron('* 30 * * * *');


let messageTTL = message.getTTL();

// same as 
messageTTL = message.getProperty(Message.PROPERTY_TTL);
```

See [Message API Reference](docs/api/message.md) for more details.


### Producer Class

Producer class is in turn responsible for producing messages. 

Each producer instance has an associated message queue and provides `produceMessage()` method which handle the
message and decides to either send it to the message queue scheduler or to immediately enqueue it for delivery.

```javascript
// filename: ./example/test-queue-producer-launch.js

'use strict';
const { Message, Producer } = require('redis-smq');

const message = new Message();

message
    .setBody({hello: 'world'})
    .setTTL(3600000)
    .setScheduledDelay(10);

const producer = new Producer('test_queue');
producer.produceMessage(message, (err) => {
   if (err) console.log(err);
   else console.log('Successfully produced')
});
```

See [Producer API Reference](docs/api/producer.md) for more details.

### Consumer Class

The Consumer class is the base class for all consumers. All consumers extends this base class and implements
`consume()` method which got called once a message is received.

Consumer classes are saved per files. Each consumer file represents a consumer class.

A consumer class may look like:

```javascript
// filename: ./example/test-queue-consumer-launch.js

'use strict';

const redisSMQ = require('redis-smq');

const Consumer = redisSMQ.Consumer;

class TestQueueConsumer extends Consumer {
    /**
     *
     * @param message
     * @param cb
     */
    consume(message, cb) {
        //  console.log('Got a message to consume:', message);
        //  
        //  throw new Error('TEST!');
        //  
        //  cb(new Error('TEST!'));
        //  
        //  const timeout = parseInt(Math.random() * 100);
        //  setTimeout(() => {
        //      cb();
        //  }, timeout);
        cb();
    }
}

TestQueueConsumer.queueName = 'test_queue';

const consumer = new TestQueueConsumer();
consumer.run();
```

To start consuming messages, a consumer needs first to be launched from CLI to connect to the Redis server 
and wait for messages: 

```text
$ node ./example/test-queue-consumer-launch.js
```

Once a message is received and processed the consumer should acknowledge the message by invoking the callback function
without arguments.

The message acknowledgment informs the message queue that the message has been successfully consumed.

If an error occurs, the message should be unacknowledged and the error should be reported to the message queue by
calling the callback function. Failed messages are re-queued and delivered again unless **message retry threshold** is
exceeded. Then the messages are moved to **dead-letter queue (DLQ)**. Each message queue has a system generated
corresponding queue called dead-letter queue where all failed to consume messages are moved to.

See [Consumer API Reference](docs/api/consumer.md) for more details.

## Performance

One key indicator about how RedisSMQ is fast and performant is Message throughput. Message throughput is the number of
messages per second that the message queue can process. 

### Scenarios

We can measure the Producer throughput and the Consumer throughput. The benchmark is composed of:

1. Measuring Producer throughput (without consumers running at the same time)
2. Measuring Consumer throughput (without producers running at the same time)
3. Measuring throughput of Producer and Consumer both running at the same time

In all scenarios messages are produced and consumed as fast as possible.


### Environment

The benchmark was performed on a VPS (4 CPU cores, 8GB RAM) running Debian 8. 

No performance tuning was performed for the VPS, neither for Redis server. Default parameters were used out of box.

The VPS was setup to run a single instance of Redis (Redis is single threaded, so more instances can boost performance).

All consumers, producers, monitor and redis server are launched from the same host.

### Results

| Scenario                                             | Producer rate (msg/sec) | Consumer rate (msg/sec) |
|-----------------------------------------------------|-------------------------|-------------------------|
| Run 1 producer instance                             | 23K+                    | 0                       |
| Run 10 producer instances                           | 96K+                    | 0                       |
| Run 1 consumer instance                             | 0                       | 13K+                    |
| Run 10 consumer instances                           | 0                       | 49K+                    |
| Run 1 producer instance and 1 consumer instance     | 22K+                    | 12K+                    |
| Run 10 producer instances and 10 consumer instances | 45K+                    | 27K+                    |
| Run 10 producer instances and 20 consumer instances | 32K+                    | 32K+                    |

Benchmarking charts are in the [screenshots folder](https://github.com/weyoss/redis-smq/tree/master/screenshots).

## Troubleshooting and monitoring

### Logs

This package is using JSON log format, thanks to [Bunyan](https://github.com/trentm/node-bunyan).

The structured data format of JSON allows analytics tools to take place but also helps to monitor and troubleshoot 
issues easier and faster.

By default all logs are disabled. Logging can affect performance (due to I/O operations). When enabled you can 
use bunyan utility to pretty format the output.

Unless configured otherwise, the standard output is the console which launched the consumer.

```text
$ node consumer | ./node_modules/.bin/bunyan
```
### Monitoring

The RedisSMQ Monitoring is an interface which let you monitor and debug your RedisSMQ server from a web browser in 
real-time.

First enable monitoring in your configuration file and provide monitoring server parameters.

Monitor server example:

```javascript
// filename: ./example/monitor.js

'use strict';

const config = require('./config');
const monitorServer = require('redis-smq').monitor(config);

monitorServer.listen(() => {
    console.log('Monitor server is running...');
});

```

Launching the server:

```text
$ node monitor.js
```
#### Monitor screenshots

Please note that the numbers shown in the screenshots are related to the Redis server configuration and the performance 
parameters of the host the server is running on!

##### Sample

Running 10 producer instances and 20 consumer instances:

![RedisSMQ Monitor](./screenshots/img_7.png)

More screenshots, could be found in the [screenshots folder](https://github.com/weyoss/redis-smq/tree/master/screenshots).

## Contributing

So you are interested in contributing to this project? Please see [CONTRIBUTING.md](https://github.com/weyoss/guidelines/blob/master/CONTRIBUTIONS.md).

## License

[MIT](https://github.com/weyoss/redis-smq/blob/master/LICENSE)