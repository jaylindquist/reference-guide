# Infrastructure

Events need to be redirected to the appropriate saga instances. To do so, some infrastructure classes are required. The most important components are the `SagaManager` and the `SagaRepository`.

## Saga Manager

Like any component that handles events, the processing is done by an event processor. However, sagas are not singleton instances handling events. They have individual life cycles which need to be managed.

Axon supports life cycle management through the `AnnotatedSagaManager`, which is provided to an event processor to perform the actual invocation of handlers. It is initialized using the type of the saga to manage, as well as a `SagaRepository` where sagas of that type can be stored and retrieved. A single `AnnotatedSagaManager` can only manage a single saga type.

When using the Configuration API, Axon will use sensible defaults for most components. However, it is highly recommended to define a `SagaStore` implementation to use. The `SagaStore` is the mechanism that 'physically' stores the saga instances somewhere. The `AnnotatedSagaRepository` \(the default\) uses the `SagaStore` to store and retrieve Saga instances as they are required.

{% tabs %}
{% tab title="Axon Configuration API" %}
```java
Configurer configurer = DefaultConfigurer.defaultConfiguration();
        configurer.eventProcessing(eventProcessingConfigurer -> eventProcessingConfigurer
                .registerSaga(MySaga.class,
                              // Axon defaults to an in-memory SagaStore,
                              // defining another is recommended
                              sagaConfigurer -> sagaConfigurer.configureSagaStore(c -> new JpaSagaStore(...))));

// alternatively, it is possible to register a single SagaStore for all Saga types:
configurer.registerComponent(SagaStore.class, c -> new JpaSagaStore(...));
```
{% endtab %}

{% tab title="Spring Boot AutoConfiguration" %}
```java
@Saga(sagaStore = "mySagaStore")
public class MySaga {...}
...
// somewhere in configuration
@Bean
public SagaStore mySagaStore() {
    return new MongoSagaStore(...); // default is JpaSagaStore
}
```
{% endtab %}
{% endtabs %}

## Saga repository and saga store

The `SagaRepository` is responsible for storing and retrieving sagas, for use by the `SagaManager`. It is capable of retrieving specific saga instances by their identifier as well as by their association values.

There are some special requirements, however. Since concurrency handling in sagas is a very delicate procedure, the repository must ensure that for each conceptual saga instance \(with an equal identifier\) only a single instance exists in the JVM.

Axon provides the `AnnotatedSagaRepository` implementation, which allows the lookup of saga instances while guaranteeing that only a single instance of the saga may be accessed at the same time. It uses a `SagaStore` to perform the actual persistence of saga instances.

The choice for the implementation to use depends mainly on the storage engine used by the application. Axon provides the `JdbcSagaStore`, `InMemorySagaStore`, `JpaSagaStore` and `MongoSagaStore`.

In some cases, applications benefit from caching saga instances. In that case, there is a `CachingSagaStore` which wraps another implementation to add caching behavior. Note that the `CachingSagaStore` is a write-through cache, which means save operations are always immediately forwarded to the backing Store, to ensure data safety.

### JpaSagaStore

The `JpaSagaStore` uses JPA to store the state and association values of sagas. Sagas themselves do not need any JPA annotations; Axon will serialize the sagas using a `Serializer` \(similar to event serialization, you can choose between an `XStreamSerializer`, `JacksonSerializer` or `JavaSerializer`, which can be set by configuring the default `Serializer` in your application. For more details, see [Serializers](../events/event-serialization.md).

The `JpaSagaStore` is configured with an `EntityManagerProvider`, which provides access to an `EntityManager` instance to use. This abstraction allows for the use of both application managed and container managed `EntityManager`s. Optionally, you can define the serializer to serialize the Saga instances with. Axon defaults to the `XStreamSerializer`.

### JdbcSagaStore

The `JdbcSagaStore` uses plain JDBC to store stage instances and their association values.
Similar to the `JpaSagaStore`, saga instances do not need to be aware of how they are stored. The store serializes the saga instances using a serializer.

You should configure the `JdbcSagaStore` with either a `DataSource` or a `ConnectionProvider`.
While not required, when initializing with a `ConnectionProvider`, it is recommended to wrap the implementation in a `UnitOfWorkAwareConnectionProviderWrapper`.
It will check the current Unit of Work for an already open database connection to ensure that all activity within a unit of work is done on a single connection.

Unlike JPA, the `JdbcSagaRepository` uses plain SQL statements to store and retrieve information.
This approach may mean that some operations depend on the database-specific SQL dialect.
It may also be that certain database vendors provide non-standard features that you would like to use.
To allow for this, you can provide your own `SagaSqlSchema`.
The `SagaSqlSchema` is an interface that defines all the operations the repository needs to perform on the underlying database.
It allows you to customize the SQL statement executed for each operation. The default is the `GenericSagaSqlSchema`.
Other implementations available are `PostgresSagaSqlSchema`, `Oracle11SagaSqlSchema` and `HsqlSagaSchema`.

> **Schema Construction**
>
> Note that Axon does not create the database schema for you out of the box.
> Neither when using Spring Boot, for example.
>
> To construct the schema, `JdbcSagaStore#createSchema` should be invoked.
> By default, this will use the `GenericSagaSqlSchema`.
> You can change the schema by configuring a different version through the `JdbcSagaStore.Builder`.

### MongoSagaStore

The `MongoSagaStore` stores the saga instances and their associations in a MongoDB database. The `MongoSagaStore` stores all sagas in a single collection in a MongoDB database. For each saga instance, a single document is created.

The `MongoSagaStore` also ensures that at any time, only a single Saga instance exists for any unique Saga in a single JVM. This ensures that no state changes are lost due to concurrency issues.

The `MongoSagaStore` is initialized using a `MongoTemplate` and optionally a `Serializer`. The `MongoTemplate` provides a reference to the collection to store the sagas in. Axon provides the `DefaultMongoTemplate`, which takes a `MongoClient` instance as well as the database name and name of the collection to store the sagas in. The database name and collection name may be omitted. In that case, they default to `"axonframework"` and `"sagas"`, respectively.

## Caching

If a database backed saga storage is used, saving and loading saga instances may be a relatively expensive operation. In situations where the same saga instance is invoked multiple times within a short time span, a cache can be especially beneficial to the application's performance.

Axon provides the `CachingSagaStore` implementation. It is a `SagaStore` that wraps another one, which does the actual storage. When loading sagas or association values, the `CachingSagaStore` will first consult its caches, before delegating to the wrapped repository. When storing information, all calls are always delegated to ensure that the backing storage always has a consistent view on the saga's state.

To configure caching, simply wrap any `SagaStore` in a `CachingSagaStore`. The constructor of the `CachingSagaStore` takes three parameters: 1. The `SagaStore` to wrap 2. The cache to use for association values 3. The cache to use for saga instances

The latter two arguments may refer to the same cache, or to different ones. This depends on the eviction requirements of your specific application.

