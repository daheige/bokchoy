# bokchoy

## Introduction

Bokchoy is a simple Go library for queueing tasks and processing them in the background with workers.

It's powered by a generic engine implementation which currently supports
[Redis](https://github.com/thoas/bokchoy/blob/master/broker_redis.go) and it's designed to
have a low barrier to entry.

It should be integrated in your web stack easily.

## Getting started

First, run a Redis server, of course:

```console
redis-server
```

Define your producer which will send tasks:

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/thoas/bokchoy"
)

func main() {
	ctx := context.Background()

	// define the main engine which will manage queues
	engine, err := bokchoy.New(ctx, bokchoy.Config{
		Broker: bokchoy.BrokerConfig{
			Type: "redis",
			Redis: bokchoy.RedisConfig{
				Type: "client",
				Client: bokchoy.RedisClientConfig{
					Addr: "localhost:6379",
				},
			},
		},
	})
	if err != nil {
		log.Fatal(err)
	}

	payload := map[string]string{
		"data": "hello world",
	}

	task, err := engine.Queue("tasks.message").Publish(ctx, payload)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(task, "has been published")
}
```

See [producer](examples/producer) directory for more information and to run it.

Now we have a producer which can send tasks to our engine, we need a worker to execute
tasks in the background:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"

	"github.com/thoas/bokchoy"
)

func main() {
	ctx := context.Background()

	engine, err := bokchoy.New(ctx, bokchoy.Config{
		Broker: bokchoy.BrokerConfig{
			Type: "redis",
			Redis: bokchoy.RedisConfig{
				Type: "client",
				Client: bokchoy.RedisClientConfig{
					Addr: "localhost:6379",
				},
			},
		},
	})
	if err != nil {
		log.Fatal(err)
	}

	engine.Queue("tasks.message").SubscribeFunc(func(r *bokchoy.Request) error {
		fmt.Println("Receive request", r)
		fmt.Println("Payload:", r.Task.Payload)

		return nil
	})

	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)

	go func() {
		for range c {
			log.Print("Received signal, gracefully stopping")
			engine.Stop(ctx)
		}
	}()

	engine.Run(ctx)
}
```

See [worker](examples/worker) directory for more information and to run it.

## Installation

Using [Go Modules](https://github.com/golang/go/wiki/Modules)

```console
go get github.com/thoas/bokchoy
```

## Advanced topics

### Custom serializer

By default the task serializer is `JSON`, you can customize it when initializing
the Bokchoy engine, it must respect the
[Serializer](https://github.com/thoas/bokchoy/blob/master/serializer.go) interface.

```go
bokchoy.New(ctx, bokchoy.Config{
    Broker: bokchoy.BrokerConfig{
        Type: "redis",
        Redis: bokchoy.RedisConfig{
            Type: "client",
            Client: bokchoy.RedisClientConfig{
                Addr: "localhost:6379",
            },
        },
    },
}, bokchoy.WithSerializer(MySerializer{}))
```

You will be capable to define a [msgpack](https://msgpack.org/), [yaml](https://yaml.org/) serializers if you want.

### Custom logger

By default the logger is disabled, you can provide a more verbose logger with options:

```go
import (
	"context"
	"fmt"
	"log"

	"github.com/thoas/bokchoy/logging"
)

func main() {
	logger, err := logging.NewDevelopmentLogger()
	if err != nil {
		log.Fatal(err)
	}

	defer logger.Sync()

    bokchoy.New(ctx, bokchoy.Config{
        Broker: bokchoy.BrokerConfig{
            Type: "redis",
            Redis: bokchoy.RedisConfig{
                Type: "client",
                Client: bokchoy.RedisClientConfig{
                    Addr: "localhost:6379",
                },
            },
        },
    }, bokchoy.WithLogger(logger))
}
```

### Worker Concurrency

By default the worker concurrency is set to `1`, you can override it based on your server
capability, Bokchoy will spawn multiple goroutines to handle your tasks.

```go
engine.Queue("tasks.message").SubscribeFunc(func(r *bokchoy.Request) error {
    fmt.Println("Receive request", r)
    fmt.Println("Payload:", r.Task.Payload)

    return nil
}, bokchoy.WithConcurrency(5))
```

### Retries

If your task handler is returning an error, the task will be marked as `failed` and retried `3 times`,
based on intervals: `60 seconds`, `120 seconds`, `180 seconds`.

You can customize this globally on the engine or when publishing a new task by using `bokchoy.WithMaxRetries`
and `bokchoy.WithRetryIntervals` options.

```go
bokchoy.WithMaxRetries(1)
bokchoy.WithRetryIntervals([]time.Duration{
	180 * time.Second,
})
```

### Timeout

By default a task will be forced to timeout if its running time exceed `180 seconds`.

You can customize this globally or when publishing a new task by using `bokchoy.WithTimeout` option:

```go
bokchoy.WithTimeout(5*time.Second)
```

### Catch events

You can catch events by registering handlers on your queue when your tasks are
starting, succeeding, completing or failing.

```go
queue := engine.Queue("tasks.message")
queue.OnStartFunc(func(r *bokchoy.Request) error {
    // we update the context by adding a value
    *r = *r.WithContext(context.WithValue(r.Context(), "foo", "bar"))

    return nil
})

queue.OnCompleteFunc(func(r *bokchoy.Request) error {
    fmt.Println(r.Context().Value("foo"))

    return nil
})

queue.OnSuccessFunc(func(r *bokchoy.Request) error {
    fmt.Println(r.Context().Value("foo"))

    return nil
})

queue.OnFailureFunc(func(r *bokchoy.Request) error {
    fmt.Println(r.Context().Value("foo"))

    return nil
})
```

### Helpers

Let's define our previous queue:

```go
queue := engine.Queue("tasks.message")
```

#### Empty the queue

```go
queue.Empty()
```

It will remove all waiting tasks from your queue.

#### Cancel a waiting task

We produce a task without running the worker:

```go
payload := map[string]string{
    "data": "hello world",
}

task, err := queue.Publish(ctx, payload)
if err != nil {
    log.Fatal(err)
}
```

Then we can cancel it by using its ID:

```go
queue.Cancel(ctx, task.ID)
```

#### Retrieve a published task from the queue

```go
queue.Get(ctx, task.ID)
```

#### Retrieve statistics from a queue

```go
stats, err := queue.Count(ctx)
if err != nil {
    log.Fatal(err)
}

fmt.Println("Number of waiting tasks:", stats.Direct)
fmt.Println("Number of delayed tasks:", stats.Delayed)
fmt.Println("Number of total tasks:", stats.Total)
```

## Contributing

* Ping us on twitter:
  * [@thoas](https://twitter.com/thoas)
* Fork the [project](https://github.com/thoas/bokchoy)
* Fix [bugs](https://github.com/thoas/bokchoy/issues)

**Don't hesitate ;)**

## Project history

Bokchoy is highly influenced by the great [rq](https://github.com/rq/rq) and [celery](http://www.celeryproject.org/).

Both are great projects well maintained but only used in a Python ecosystem.

