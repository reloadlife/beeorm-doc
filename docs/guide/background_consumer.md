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
   validatedRegistry, err := registry.Validate(context.Background())
    if err != nil {
        panic(err)
    }
    engine := validatedRegistry.CreateEngine(context.Background())
    consumer := beeorm.NewBackgroundConsumer(engine)
    consumer.Digest() // code is blocked here
}

```

This script uses another BeeORM feature called `Event Consumer`. 
You will learn more about on [next pages](/TODO). `Event Consumer` intefrace
provides you many useful methods that you can use to configure `Background Consumer`:

 * [DisableLoop()](/TODO)
 * [SetLimit()](/TODO)
 * [SetHeartBeat()](/TODO)
 * [SetErrorHandler()](/TODO)