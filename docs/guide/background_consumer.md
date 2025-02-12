# Background consumer

Many operations in BeeORM, that we will explain later on next pages, require
some background asynchronous tasks to be executed. To use these features you must
run at least one goroutine or go program that is executing `beeorm.BackgroundConsumer`:

```go{13-14}
package main

import "github.com/latolukasz/beeorm"

func main() {
   registry := beeorm.NewRegistry()
   // ... register services in registry
   validatedRegistry, deferF, err := registry.Validate()
    if err != nil {
        panic(err)
    }
    defer deferF()
    engine := validatedRegistry.CreateEngine()
    consumer := beeorm.NewBackgroundConsumer(engine)
    consumer.Digest(context.Background()) // code is blocked here
}

```

This script uses another BeeORM feature called `Event Consumer`. 
You will learn more about on [next pages](/guide/event_broker.html#consuming-events).

## Defining backround consumer redis pools
By default, all features related to `Background Consumer` create redis streams in
``default`` redis pool name. You can define any pool name you want:


<code-group>
<code-block title="code">
```go
registry := beeorm.NewRegistry()
registry.RegisterRedis("192.123.11.12:6379", "", 0, "another_pool")
// lazy flush
registry.RegisterRedisStream("orm-lazy-channel", "another_pool", []string{"orm-async-consumer"})
// logs tables
registry.RegisterRedisStream("orm-log-channel", "another_pool", []string{"orm-async-consumer"})
// redis search indexer
registry.RegisterRedisStream("orm-redis-search-channel", "another_pool", []string{"orm-async-consumer"})
// redis streams garbage collector
registry.RegisterRedisStream("orm-stream-garbage-collector", "another_pool", []string{"orm-async-consumer"})
```
</code-block>

<code-block title="yaml">
```yml{4,5}
another_pool:
    redis: 192.123.11.12:6379
    streams:
        orm-lazy-channel:
          - orm-async-consumer
        orm-log-channel:
          - orm-async-consumer
        orm-redis-search-channel:
          - orm-async-consumer
        orm-stream-garbage-collector:
          - orm-async-consumer
```
</code-block>
</code-group>
