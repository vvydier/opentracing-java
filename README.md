[![Build Status][ci-img]][ci] [![Coverage Status][cov-img]][cov] [![Released Version][maven-img]][maven]

# OpenTracing API for Java

This library is a Java platform API for OpenTracing.

## Required Reading

In order to understand the Java platform API, one must first be familiar with
the [OpenTracing project](http://opentracing.io) and
[terminology](http://opentracing.io/documentation/pages/spec.html) more specifically.

## Status

This project has a working design of interfaces for the OpenTracing API. There
is a [MockTracer](https://github.com/opentracing/opentracing-java/tree/master/opentracing-mock)
to facilitate unit-testing of OpenTracing Java instrumentation.

Packages are deployed to Maven Central under the `io.opentracing` group.

## Usage

### Initialization

Initialization is OpenTracing-implementation-specific. Generally speaking, the pattern is to initialize a `Tracer` once
for the entire process and to use that `Tracer` for the remainder of the process lifetime. The
[GlobalTracer](https://github.com/opentracing-contrib/java-globaltracer) repository provides a helper for singleton
access to the `Tracer` as well as `ServiceLoader` support for OpenTracing Java implementations.

### "Active" `Span`s and within-process prapagation

For any execution context or Thread, at most one `Span` may be "active". Of course there may be many other `Spans` involved with the execution context which are (a) started, (b) not finished, yet (c) not "active": perhaps they are waiting for I/O, blocked on a child Span, or otherwise off the critical path.
 
It's inconvenient to pass an active `Span` from function to function manually, so OpenTracing provides a `ActiveSpanHolder` abstraction to provide access to the active `Span` and to capture it for reactivation in other execution contexts (e.g., in an async callback).

Every `Tracer` implementation _must_ provide access to a `ActiveSpanHolder` (typically provided at `Tracer` initialization time). The `ActiveSpanHolder` in turn exposes the active `Span`, like so:

```
    io.opentracing.Tracer tracer = ...;
    ...
    Span active = tracer.holder().activeSpan();
    if (active != null) {
        active.log("...");
    }
```

### Starting a new Span

The simplest case looks like this:

```
    io.opentracing.Tracer tracer = ...;
    ...
    try (Span span = tracer.buildSpan("someWork").start()) {
        // (do things / record data to `span`)
    }
```

Or, to take advantage of `ActiveSpanHoldar` and automatic intra-process propagation, like this:

```
    io.opentracing.Tracer tracer = ...;
    ...
    try (ActiveSpanHoldar.Continuation cont = tracer.buildSpan("someWork").startAndActivate()) {
        Span span = cont.span();
        // (do things / record data to `span`)
    }
```


**If there is an active `Span` per `ActiveSpanHoldar`, it will act as the parent to any newly started `Span`** unless the programmer provides an explicit reference at `buildSpan` time, like so:

```
    io.opentracing.Tracer tracer = ...;
    ...
    Span span = tracer.buildSpan("someWork").asChildOf(otherSpanContext).start();
```

### Deferring asynchronous work

Consider the case where a `Span`'s lifetime logically starts in one execution context and ends in another. For instance, the intra-Span timing breakdown might look like this:

```
[ ServiceHandlerSpan                                 ]
|-FunctionA-|-----waiting on an RPC------|-FunctionB-|
            
------------------------------------------------> time
```

The `ServiceHandlerSpan` is _active_ when it's running FunctionA and FunctionB, and inactive while it's waiting on an RPC (presumably modelled as its own Span, though that's not the concern here).

**The `ActiveSpanHolder` makes it easy to capture the Span and execution context in `FunctionA` and re-activate it in `FunctionB`.** These are the steps:

1. In the method/function that *allocates* the closure/`Runnable`/`Future`/etc, call `ActiveSpanHolder#captureActive()` to obtain a `ActiveSpanHolder.Continuation`
2. In the closure/`Runnable`/`Future`/etc itself, pair calls to `ActiveSpanHolder.Continuation#activate` and `ActiveSpanHolder.Continuation#deactivate`

In practice, the latter is most fluently accomplished through the use of an OpenTracing-aware `ExecutorService` and/or `Runnable`/`Callable` adapter; they can factor all of the above.

#### Reference counting with `ActiveSpanHolder.Continuation`

When an `ActiveSpanHolder.Continuation` is created (either via `Tracer.SpanBuilder#startAndActivate`, `ActiveSpanHolder#capture(Span)`, or `ActiveSpanHolder.Continuation#capture()`), the reference count for the `Continuation`'s `Span` increments. Whenever a `Continuation` for a `Span` `deactivate()`s, the reference count decrements. When the reference count decrements to zero, the associated `Span`'s `finish()` method is invoked automatically.

In practice, this means that, when used as designed, the programmer should not need to invoke `Span#finish()` manually, but rather should let `ActiveSpanHolder` and `ActiveSpanHolder.Continuation` invoke `finish()` as soon as the last active or deferred `ActiveSpanHolder.Continuation` is deactivated.

# Development

This is a maven project, and provides a wrapper, `./mvnw` to pin a consistent
version. For example, `./mvnw clean install`.

This wrapper was generated by `mvn -N io.takari:maven:wrapper -Dmaven=3.5.0`

## Building

Execute `./mvnw clean install` to build, run tests, and create jars.

## Contributing

See [Contributing](CONTRIBUTING.md) for matters such as license headers.


  [ci-img]: https://travis-ci.org/opentracing/opentracing-java.svg?branch=master
  [ci]: https://travis-ci.org/opentracing/opentracing-java
  [cov-img]: https://coveralls.io/repos/github/opentracing/opentracing-java/badge.svg?branch=master
  [cov]: https://coveralls.io/github/opentracing/opentracing-java?branch=master
  [maven-img]: https://img.shields.io/maven-central/v/io.opentracing/opentracing-api.svg?maxAge=2592000
  [maven]: http://search.maven.org/#search%7Cga%7C1%7Copentracing-api
