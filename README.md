![travis](https://img.shields.io/travis/streamingpool/streamingpool-core/master.svg)
![release](https://img.shields.io/github/release/streamingpool/streamingpool-core.svg)
![license](https://img.shields.io/github/license/streamingpool/streamingpool-core.svg)
[![codecov](https://codecov.io/gh/streamingpool/streamingpool-core/branch/master/graph/badge.svg)](https://codecov.io/gh/streamingpool/streamingpool-core)
[![Codacy code quality](https://api.codacy.com/project/badge/Grade/b398bb4734f64bd28767234b88a75c93)](https://www.codacy.com/app/tensorics/streamingpool-core?utm_source=github.com&utm_medium=referral&utm_content=streamingpool/streamingpool-core&utm_campaign=Badge_Grade)

## Getting Started
In order to use the Streaming Pool, just get the latest version from [Maven Central](https://search.maven.org/#search%7Cga%7C1%7Ca%3A%22streamingpool-core%22).

Maven:
```xml
<dependency>
    <groupId>org.streamingpool</groupId>
    <artifactId>streamingpool-core</artifactId>
    <version>X.Y.Z</version>
</dependency>
```

Gradle:
```groovy
compile 'org.streamingpool:streamingpool-core:X.Y.Z'
```

It is possible to check the examples by cloning this repository and checking the `src/examples` folder. 
There, core functionality of the Streaming Pool project are explained.

Keep in mind that Streaming Pool assumes that the [Spring Framework](http://projects.spring.io/spring-framework/) 
is used for managing the application Beans.

## Motivation
When connecting together heterogeneous and complex systems, it is not easy to exchange data between components. Streams of data are successfully used in industry in order to overcome this problem, especially in the case of "live" data. Streams are a specialization of the Observer design pattern and they provide asynchronous and non-blocking data flow.

The ongoing effort of the http://reactivex.io/[ReactiveX] initiative is one example that demonstrates how demanding this technology is even for big companies. Bridging the discrepancies of different technologies with common interfaces is already done by the http://www.reactive-streams.org/[Reactive Streams] initiative and, in the JVM world, via https://github.com/reactive-streams/reactive-streams-jvm[reactive-streams-jvm] interfaces.

Streaming Pool is a framework for providing and discovering reactive streams. Through the mechanism of dependency injection provided by the http://projects.spring.io/spring-framework/[Spring Framework], Streaming Pool provides a so called `DiscoveryService`. This object can discover and chain streams of data that are technologically agnostic, through the use of instances of the `StreamId` interface. The stream to be discovered must be present in the Streaming Pool system (by providing it using the `ProvidingService`) or it must be possible to create it (using one of the registered `StreamFactory`). In the latter case, the stream is lazily created on demand.
The application (client) that uses a stream does not need to know which is the source of the information, it may be a component of the application itself or a distributed system. In this way, it is possible to create truly decoupled systems that are resilient to changes and are easy to mock or test.

## Introduction
A common use case in modern Software applications is to access the values from a particular source. In the context of Internet of Things the source can be a sensor or a device. In business applications the source can be a real time indicator or application state. At CERN a common problem is to listen for devices values of the control system of different machines.

All these problems can be solved using the well-known Observable-Observer pattern or a Reactive Stream of the data. Streaming Pool assumes you want to use a Reactive Stream for accessing your data.

There are technologies that already solve this problem, such as https://projectreactor.io/[Project Reactor] and https://github.com/ReactiveX/RxJava[RxJava]. They provide stream creation, manipulation and subscription capabilities and they are great. Streaming Pool sits on top of these technologies.

The goal of Streaming Pool is to manage, reuse and share Reactive Streams, locally (in the same JVM) or remotely (distributed environment). The only required knowledge is the ID or `StreamId` that describes the Reactive Stream to obtain.

### StreamId
A `StreamId` is an object that implements the `StreamId<T>` interface. The generic type indicates the type of the data in the Reactive Stream. For each `StreamId<T>` there is a corresponding `Publisher<T>` and we enforce to have a consistent type by design.

Note that the `StreamId` interface is a marker interface, it is used as a key to distinguish `Publisher` instances in the Streaming Pool. Depending on your needs, the `StreamId` can be used to carry information about the characteristics of the Reactive Stream you want to get (especially useful in combination with a `StreamFactory`).

__CAUTION__

> Since the `StreamId` acts as key in the Streaming Pool for the corresponding `Publisher` it
 *must implement* `equals(Object o)` and `hashCode()` methods. If it is not the case, the `Publisher`
  in the Streaming Pool will not be reused and the behavior could be unpredictable.


### Discover a stream
```java
class TemperatureAdapter {
    @Autowired
    private DiscoveryService discoveryService;

    public Flowable<Double> temperaturesOf(Building building, Floor floor) {
        StreamId<Double> temperatureStreamId = TemperatureStreamId.builder()
          .ofBuilding(building)
          .ofFloor(floor)
          .build();

        Publisher<Double> temperatureStream = discoveryService.discover(temperatureStreamId);
        return Flowable.fromPublisher(temperatureStream);
    }
}
```
In this example, the `Publisher` (Reactive Stream) related to the specified `deviceStreamId` is then returned. Behind the scenes, the Streaming Pool engine will retrieve the proper `Publisher`.

__NOTE__

> Keep in mind that you can query for the same ID multiple times and you will get the same instance of 
the `Publisher` (like in a `Map<StreamId<T>, Publisher<T>>`).

### Provide a stream
```
class TemperatureProvider {
    @Autowired
    private ProvidingService providingService;

    public void provideTemperatureStream(Building building, Floor floor, Flowable<Double> temperatures) {
      StreamId<Double> temperatureStreamId = TemperatureStreamId.builder()
        .ofBuilding(building)
        .ofFloor(floor)
        .build();

        providingService.provide(temperatureStreamId, temperatures);
    }
}
```
In the case you want to provide a Reactive Stream, you can easily do so by using the `ProvidingService` interface. It has a single method that will register in the Streaming Pool system the given `Publisher` associated to the given `StreamId`. Later on, you can discover the stream by querying a `DiscoveryService` with the same `StreamId`.

__NOTE__
>It is currently not possible to remove a Reactive Stream from the Streaming Pool. 
This feature is scheduled to be implemented, but it is not part of the current development.

### How to lazily create Reactive Streams
You do not have to explicitly provide `Publisher` instances in the Streaming Pool. A common way of creating Reactive Streams is through the use of `StreamFactory`. A `StreamFactory` is a mechanism that is used to create a `Publisher` from a given `StreamId`. When looking for a `StreamId` that has not been provided using the `ProvidingService`, the framework checks if any registered `StreamFactory` is able to create it.

Using this mechanism, a `Publisher` is created on-demand (lazily) if it is not already present in the Streaming Pool system. A Stream Factory is a class that implements the interface `StreamFactory`.

#### Stream factory
```java
<T> Optional<Publisher<T>> create(StreamId<T> id, DiscoveryService discoveryService);
```

A `StreamFactory` needs to implement the `create(...)` method in which they have to:

1. decide if it can create a `Publisher` for the given `StreamId`
2. actually create the `Publisher` and return it

During the stream creation, you have access to the `DiscoveryService` in the case you need to lookup other Reactive Streams. You should be aware though that circular dependencies during stream creation are detected and the discovery method will throw accordingly.

*`StreamId` discovery is not thread-safe*, therefore it is *forbidden* to use different threads inside a `StreamFactory#create` method. This case is checked and Streaming Pool will throw an exception.

__NOTE__
>In case the `StreamFactory` is not able to create the current `StreamId`, by convention it must return an empty `Optional`.

__IMPORTANT__
> By method signature, the type of the `StreamId` and the type of the produced `Publisher` must match. Often, you will have your own types of `StreamId`, so after proper checking you can cast to your own instance of `StreamId`. Again, after the creation is ok to cast again the `Publisher` to a `Publisher<T>` to satisfy the Java compiler. This trick is needed, mostly, because of the generics implementation in Java.

In order to use your `StreamFactory`, you have to register it. Streaming Pool makes extensive use of Spring dependency injection, and it collects all the objects that are implementing the `StreamFactory` interface in the context. Those Beans will be then registered in the Streaming Pool and they will be used in the discovery process if needed. Therefore, you just have to provide a Bean for your factories.

### How discovery works
One of the key feature of Streaming Pool is the discovery of a Reactive Stream using the `DiscoveryService`.

The discovery can be summarized by the following pseudo-code.
```
function discover(SteamId id)

    if streamingPoolContains(id) <1>
        return getStreamFor(id)

    if not streamFactoriesCanCreate(id) <2>
        throws exception

    return streamFactoriesCreate(id) <3>
```
* <1> check if the `StreamId` is already present in the Streaming Pool and return it.
* <2> if the stream cannot be created by any factory, then an error is thrown. In this case, make sure you are registering your `StreamFactory` correctly.
* <3> a `StreamFactory` is able to create the Reactive Stream, so it the stream is created and registered in the Streaming Pool.

## Examples
It is possible to find examples of the Streaming Pool features in the folder `src/examples` in the repository source code. 
The examples are expressed as JUnit tests and they can be run and modified. The goal is to provide a quickstart for understanding how Streaming Pool works.

We assume that you have a basic understanding of the [Spring Framework](http://projects.spring.io/spring-framework/) 
dependency injection using annotations.
