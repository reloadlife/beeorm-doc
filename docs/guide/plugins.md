# Plugins

BeeORM can be enhanced with new functionality through the use of plugins. Currently, these plugins only allow for partial extension of BeeORM's features, but in upcoming releases, it will be possible to extend all of BeeORM's functionalities.

## Enabling Plugins

Activating plugins in BeeORM is straightforward. Simply register the plugin using the `RegisterPlugin()` method:

```go{10}
package main

import (
    "github.com/latolukasz/beeorm/v2"
    "github.com/latolukasz/beeorm/v2/plugins/log_tables"
)

func main() {
  registry := beeorm.NewRegistry()
  registry.RegisterPlugin(log_tables.Init())
}
```

BeeORM offers several built-in plugins that can be found in the [Plugins](/plugins/) section.

## Creating a BeeORM Plugin

You can create your own custom BeeORM plugin by implementing the following interface:

```go
type Plugin interface {
	GetCode() string
}
```

The GetCode function returns a unique name that identifies your plugin. It is recommended to include the go module name in the plugin name, such as `github.com/latolukasz/beeorm/my_debugger`, to prevent collisions with other plugins created by other developers.

Here's an example of a basic plugin called `my_debugger`:

```go
package my_debugger

const PluginCode = "github.com/me/my_project/my_debugger"

type MyDebuggerPlugin struct{}

func (p *MyDebuggerPlugin) GetCode() string {
	return PluginCode
}
```

Once you have created a basic plugin, you can implement one or many of the BeeORM plugin interfaces defined [here](https://github.com/latolukasz/beeorm/blob/v2/plugin.go) to add custom functionality.

### PluginInterfaceInitRegistry

```go
type PluginInterfaceInitRegistry interface {
	PluginInterfaceInitRegistry(registry *Registry)
}
```

The `PluginInterfaceInitRegistry` interface is executed when the plugin is registered in the BeeORM Registry.

:::tip
The code for plugins is executed in the same order in which they were registered in BeeORM.
:::

### PluginInterfaceInitEntitySchema

```go
type PluginInterfaceInitEntitySchema interface {
	InterfaceInitEntitySchema(schema SettableEntitySchema, registry *Registry) error
}
```

The `PluginInterfaceInitEntitySchema` interface is executed for every Entity when the [registry.Validate()](/guide/validated_registry.html#validating-the-registry)  method is called. You have access to the `beeorm.Registry` and the `beeorm.SettableEntitySchema` object, which allows you to save additional settings in the [Entity Schema](/guide/validated_registry.html#entity-schema) using the `SetPluginOption()` method.

For example:

```go{12-13}
package my_debugger

const PluginCode = "github.com/me/my_project/my_debugger"

type MyDebuggerPlugin struct{}

func (p *MyDebuggerPlugin) GetCode() string {
	return PluginCode
}

func (p *MyDebuggerPlugin) InterfaceInitEntitySchema(schema SettableEntitySchema, _ *Registry) error {
	schema.SetPluginOption(PluginCode, "my-option-1", 200)
	schema.SePlugintOption(PluginCode, "my-option-2", "Hello")
	return nil
}
```

Note that the `schema.SetPluginOption` method requires the plugin code name as its first argument, to prevent overrides of options with the same name from other plugins. The Entity Schema options can be easily accessed through the `GetPluginOption...` methods.

For example:

```go
entitySchema := validatedRegistry.GetEntitySchema(carEntity)
entitySchema.GetPluginOption(PluginCode, "my-option-1") // int(200)
entitySchema.GetPluginOption(PluginCode, "my-option-2") // "Hello"
entitySchema.GetPluginOption(PluginCode, "missing-key") // nil
```

### PluginInterfaceTableSQLSchemaDefinition

```go
type PluginInterfaceTableSQLSchemaDefinition interface {
	PluginInterfaceTableSQLSchemaDefinition(engine Engine, sqlSchema *TableSQLSchemaDefinition) error
}
```


This interface is executed during Entity MySQL [schema update](/guide/schema_update.html). When this interface is called, you have access to the `TableSQLSchemaDefinition` object, which provides information about the Entity schema definition and the current MySQL table schema (if the table already exists in the database):

```go
func (p *MyDebuggerPlugin) PluginInterfaceTableSQLSchemaDefinition(_ beeorm.Engine, sqlSchema *beeorm.TableSQLSchemaDefinition) error {
    sqlSchema.EntitySchema // entity schema that is currently under validation
    sqlSchema.EntityColumns // Entity columns SQL definitions
    sqlSchema.EntityIndexes // Entity SQL indexes definitions
    sqlSchema.DBTableColumns // columns SQL definitions from current table
    sqlSchema.DBIndexes // SQL index definitions from current table
    sqlSchema.DBCreateSchema // SQL `CREATE TABLE..` definition from current table
    sqlSchema.DBEncoding // encoding from current table
    sqlSchema.PreAlters // alters that should be executed at the beginning of schema update
    sqlSchema.PostAlters // alters that should be executed at the end of schema update
```

Here's an example that demonstrates how to implement a plugin that adds a `IP VARCHAR(15)` column to every table:

```go
func (p *MyDebuggerPlugin) PluginInterfaceTableSQLSchemaDefinition(_ beeorm.Engine, sqlSchema *beeorm.TableSQLSchemaDefinition) error {
 hasIPColumn := false 
 name := "IP"
 definition := "`+name+` VARCHAR(15)"
 for _, column := range sqlSchema.EntityColumns {
    if column.ColumnName == name {
        if column.Definition != definition {
            return fmt.Errorf("column %s  with wrong definition `%s` already exist", name, definition)
        }
        hasIPColumn = true
        break
    }
 }
 if !hasIPColumn {
    ipColumnDefinition := &beeorm.ColumnSchemaDefinition{ColumnName: name, Definition: definition}
    sqlSchema.EntityColumns = append(sqlSchema.EntityColumns, ipColumnDefinition)
 }
}
```

In the example above, the `PluginInterfaceTableSQLSchemaDefinition` method first checks if the IP column with the correct definition already exists. If it does not, the method adds the IP column definition to the `TableSQLSchemaDefinition` object's EntityColumns field. This effectively adds the IP column to the MySQL table when the schema update is executed.

### PluginInterfaceEntityFlushing 

```go
type PluginInterfaceEntityFlushing interface {
	PluginInterfaceEntityFlushing(engine Engine, event EventEntityFlushing)
}
```

The `PluginInterfaceEntityFlushing` interface is called every time an Entity is flushed prior to the execution of the SQL query in the MySQL database. The `EventEntityFlushing` object contains information about the changes, such as the type of action (Insert, Update, or Delete), the entity name, ID, and the values of the fields before and after the SQL query.

Additionally, the EventEntityFlushing object provides a method `SetMetaData()` which allows you to store extra parameters in the flush entity event that can be used in subsequent plugin interfaces, such as [PluginInterfaceEntityFlushed](/guide/plugins.html#plugininterfaceentityflushed).

```go
func (p *MyDebuggerPlugin) PluginInterfaceEntityFlushing(engine Engine, event EventEntityFlushing) {
    event.Type() // beeorm.Insert or beeorm.Update or beeorm.Delete
    event.EntityName() // flushed entity name, for instance "package.CarEntity"
    event.EntityID() // flushed entity ID, zero when  event.Type() is beeorm.Insert
    event.Before() // map of changed fields values before SQL query. nil when Action is "beeorm.Insert"
    event.After() // map of changed fields values after SQL query. nil when Action is "beeorm.Delete"
    event.SetMetaData("meta-1", "value")
}
```

### PluginInterfaceEntityFlushed

```go
type PluginInterfaceEntityFlushed interface {
	PluginInterfaceEntityFlushed(engine Engine, event EventEntityFlushed, cacheFlusher FlusherCacheSetter)
}
```

This interface is executed every time an Entity is flushed in the MySQL database after the SQL query has been executed but before any cache-related operations (e.g. deleting the Entity cache in Redis) are carried out.

The `EventEntityFlushed` object holds all information about the changes made to the Entity.

```go
func (p *MyDebuggerPlugin) PluginInterfaceEntityFlushed(engine Engine, event EventEntityFlushed, cacheFlusher FlusherCacheSetter) {
    event.Type() // beeorm.Insert or beeorm.Update or beeorm.Delete
    event.EntityName() // flushed entity name, for instance "package.CarEntity"
    event.EntityID() // flushed entity ID
    event.Before() // map of changed fields values before SQL query. nil when Action is "beeorm.Insert"
    event.After() // map of changed fields values after SQL query. nil when Action is "beeorm.Delete"
    event.MetaData() // meta data set in `PluginInterfaceEntityFlushing`
}
```

For example:

```go
package my_package

car := &CarEntity{Name: "BMW", Year: 2006}
engine.FLush(car)
// then
event.Type() // beeorm.Insert
event.EntityName() // my_package.CarEntity
event.EntityID() // 1
event.Before() // nil
event.After() // {"Name": "Bmw", "Year": "2006"}

car.Year = 2007
engine.Flush(car)
// then
event.Type() // beeorm.Update
event.EntityName() // my_package.CarEntity
event.EntityID() // 1
event.Before() // {"Year": "2006"}
event.After() // {""Year": "2007"}

engine.Delete(car)
engine// then
event.Type() // beeorm.Delete
event.EntityName() // my_package.CarEntity
event.EntityID() // 1
event.Before() // {"Name": "BMW", "Year": "2007"}
event.After() // nil
```

The last argument, `FlusherCacheSetter`, is used to add extra cache operations to Redis or the local cache that should be executed after the Entity is updated in the database. BeeORM groups all cache operations in an optimized way by using pipelines.

For example:

```go
func (p *MyDebuggerPlugin) PluginInterfaceEntityFlushed(engine Engine, event EventEntityFlushed, cacheFlusher FlusherCacheSetter) {
    cacheFlusher.GetRedisCacheSetter("default").Del("my-key")
    cacheFlusher.PublishToStream("my-stream", "hello")
}
```

### PluginInterfaceEntitySearch

```go
type PluginInterfaceEntitySearch interface {
	PluginInterfaceEntitySearch(engine Engine, schema EntitySchema, where *Where) *Where
}
```

The PluginInterfaceEntitySearch interface is executed every time a user searches for entities using BeeORM's [search feature](/guide/search.html). It allows you to modify the search query by returning a new `beeorm.Where` object.

Here is an example implementation of this interface:

```go
func (p *MyDebuggerPlugin) PluginInterfaceEntitySearch(engine Engine, schema EntitySchema, where *Where) *Where {
    if schema.GetEntityName() == "myproject.UserEntity" {
        return beeorm.NewWhere("`Active` == 1 AND " + where.String(), where.GetParameters()...)
    }
    return where
}
```

In this example, the `PluginInterfaceEntitySearch` method checks if the entity being searched for is a UserEntity. If it is, the method returns a new Where object that includes an additional condition to filter out inactive users. If the entity being searched for is not a UserEntity, the original `Where` object is returned unmodified.

## Engine Plugin Options

The `SetPluginOption` and `GetPluginOption` methods of the `beeorm.Engine` allow you to store and retrieve extra options in the engine.

Here's an example of how you can use these methods:

```go
engine.SetPluginOption(PluginCode, "option-1", "value")
val := engine.GetPluginOption(PluginCode, "options-1") // "value"
```