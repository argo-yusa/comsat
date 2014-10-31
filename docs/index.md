---
layout: default
title: Comsat
description: "Comsat integrates lightweight threads and actors with the JVM web stack."
---

# Overview

COMSAT (or Comsat) is a set of open source libraries that integrate [Quasar](http://puniverse.github.io/quasar/) with various web or enterprise technologies (like HTTP services and database access). With Comsat, you can write web applications that are scalable and performant while, at the same time, are simple to code and maintain.

Comsat is not a web framework. In fact, it does not add new APIs at all (with one exception, Web Actors, mentioned later). It provides implementation to popular (and often, standard) APIs like Servlet, JAX-RS, and JDBC, that can be called within Quasar fibers.

Comsat does provide one new API that you may choose to use: [Web Actors](manual/webactors.html). Web actors let you define a Quasar actor that receives and responds to HTTP requests and web socket messages. The Web Actors API is rather minimal, and is intended to do one job and do it well: simplify two-way communication between your server and the client.

[Parallel Universe]: http://paralleluniverse.co

## News

### July 23, 2014

COMSAT [0.2.0](https://github.com/puniverse/comsat/releases/tag/v0.2.0) has been released.

### January 22, 2014

COMSAT [0.1.0](https://github.com/puniverse/comsat/releases/tag/v0.1.0) has been released.

# Getting Started

### System requirements

Java 7 is required to use COMSAT.

### Using Maven {#maven}

First, you must add the `quasar-core` dependency:

~~~ xml
<dependency>
    <groupId>co.paralleluniverse</groupId>
    <artifactId>quasar-core</artifactId>
    <version>{{site.version}}</version>
</dependency>
~~~

or, for JDK8:

~~~ xml
<dependency>
    <groupId>co.paralleluniverse</groupId>
    <artifactId>quasar-core</artifactId>
    <version>{{site.version}}</version>
    <classifier>jdk8</classifier>
</dependency>
~~~

Then add those Comsat modules that you'd like to use:

~~~ xml
<dependency>
    <groupId>co.paralleluniverse</groupId>
    <artifactId>ARTIFACT</artifactId>
    <version>{{site.version}}</version>
</dependency>
~~~

where `ARTIFACT` is:

* `comsat-servlet` – Servlet integration for defining fiber-per-request servlets.
* `comsat-jersey-server` – [Jersey server](https://jersey.java.net/) integration for defining REST services.
* `comsat-dropwizard` – [Dropwizard](http://dropwizard.io/) integration including Jersey, ApacheHttpClient and JDBI.
* `comsat-jax-rs-client` – [JAX-RS client](https://jersey.java.net/documentation/latest/client.html) integration for calling HTTP services.
* `comsat-httpclient` – [ApacheHttpClient](http://hc.apache.org/httpcomponents-client-ga/) integration for calling HTTP services.
* `comsat-retrofit` – [Retrofit](http://square.github.io/retrofit/) integration.
* `comsat-jdbi` – [JDBI](http://jdbi.org/) integration for using the JDBI API in fibers.
* `comsat-jdbc` – JDBC integration for using the JDBC API in fibers.
* `comsat-jooq` – [jOOQ](http://www.jooq.org/) integration for using the jOOQ API in fibers.
* `comsat-mongodb-allanbank` – MongoDB integration for using the [allanbank API](http://www.allanbank.com/mongodb-async-driver/index.html)
* `comsat-actors-api` – the Web Actors API
* `comsat-actors-servlet` – Enables HTTP and WebSocket (JSR-356) usage through Web Actors API
* `comsat-tomcat-loader` – Enables using Comsat in Tomcat container without the need of javaAgent
* `comsat-jetty-loader` – Enables using Comsat in Jetty container without the need of javaAgent

### Enabling Comsat

Comsat runs code in [Quasar](http://docs.paralleluniverse.co/quasar/) fibers, which rely on bytecode instrumentation. This instrumentation is done in one of three ways: via a Java agent that must be loaded into the Servlet container; with a custom class-loader available for Tomcat and Jetty; or at compilation time.

AOT instrumentation is eplained in the [Quasar documentation](http://docs.paralleluniverse.co/quasar/index.html#instruemtnation).

#### The Java Agent

To use the Java agent, the following must be added to the java command line (or use your favorite build tool to add this as a JVM argument) when launching the process (Servlet container if you're deploying your app as a WAR file):

~~~ sh
-javaagent:path-to-quasar-jar.jar
~~~

#### In Tomcat

If you're using Tomcat as your Servlet container, you have the option to use a custom class-loader instead of the Java agent. You'll need to put `comsat-tomcat-loader-{{site.version}}.jar` (or, for JDK8, `comsat-tomcat-loader-{{site.version}}-jdk8.jar`) into Tomcat's `lib` directory.

Then, include the following in your webapp's `META-INF/context.xml`:

~~~ xml
{% include_snippet loader ./comsat-test-war/src/main/webapp/META-INF/context.xml %}
~~~

#### In Jetty

If you're using Jetty as your Servlet container, you have the option to use a custom class-loader instead of the Java agent. You'll need to put `comsat-jetty-loader-{{site.version}}.jar` (or, for JDK8, `comsat-jetty-loader-{{site.version}}-jdk8.jar`) into Jetty's `lib` directory.

Then, include a `<Set name="classLoader">` tag in your webapp's context xml:

~~~ xml
{% include_snippet context xml ./comsat-jetty-loader/src/test/resources/webapps/dep.xml %}
~~~

### Building Comsat {#build}

Clone the repository:

    git clone https://github.com/puniverse/comsat.git

and run:

    ./gradlew

Or, if you have gradle installed, run:

    gradle

# User Manual

{% capture javadoc %}{{site.baseurl}}/javadoc/co/paralleluniverse{% endcapture %}

## Comsat Integration

### Servlets

Comsat integrates with JavaEE servlets and enables you to write servlets that can scale to many concurrent visitors, even if servicing each requests takes a very long time, or requires calling many other services. Under the hood, Comsat does this by turning each servlet request into an asynchronous request, and then services each request on a separate [fiber](http://puniverse.github.io/quasar/manual/core.html#fibers). Calls to other web services or to a database are fiber- rather than thread-blocking. As a result, Comsat can serve many thousands of concurrent requests with only a handful of OS threads. You, on the other hand, don't need to adopt a cumbersome asynchronous programming model. You can write the servlet code as you normally would, making synchronous (fiber-blocking) calls, provided that you use Comsat implementations.

To write a Comsat (fiber-per-request) servlet, simply extend [`FiberHttpServlet`]({{javadoc}}/fibers/servlet/FiberHttpServlet.html) rather than the usual `javax.servlet.HttpServlet`, and either annotate it with `@WebServlet` or declare it in `web.xml`. Note how the `service` and all the `doXXX` methods are [suspendable](http://puniverse.github.io/quasar/manual/core.html#fibers) (they all `throw SuspendExecution`).

You can deploy your servlet as you normally would, either as a WAR file, or in an embedded servlet container.

It is recommended that you then configure your servlet container to limit the number of threads in its thread pool to some small number, as all these threads do is create the fiber (which runs in the fiber thread pool) and return.

Example:

~~~ java
{% include_snippet FiberHttpServlet example ./comsat-servlet/src/test/java/co/paralleluniverse/fibers/servlet/FiberHttpServletTest.java %}
~~~
Then you can simply add it as a regular servlet to you favorite servlet containter:

~~~ java
{% include_snippet servlet registration ./comsat-servlet/src/test/java/co/paralleluniverse/fibers/servlet/FiberHttpServletTest.java %}
~~~

To learn about writing servlets, you can refer to the [Java Servlets tutorial](http://docs.oracle.com/javaee/7/tutorial/doc/servlets.htm).

### REST Services

You can easily create Comsat REST services with the [JAX-RS API](https://jax-rs-spec.java.net/nonav/2.0/apidocs/index.html), the standard Java REST service API. Comsat integrates with [Jersey](https://jersey.java.net/), the reference JAX-RS implementation.

All you need to do in order to enjoy Comsat's scalabilty, is replace the line

~~~ xml
<servlet-class>org.glassfish.jersey.servlet.ServletContainer</servlet-class>
~~~

in your `web.xml` file, which is how you would normally use Jersey in a Servlet container, with:

~~~ xml
<servlet-class>co.paralleluniverse.fibers.jersey.ServletContainer</servlet-class>
~~~

Your resource methods (the ones you annotate with `@GET`, `@PUT`, `@POST` etc.) can now be made suspendable by declaring `throws SuspendExecution`. Comsat would then run each request in a fiber. Your resource methods are free to use Comsat's JDBC implementation, or Comsat's JAX-RS client.

Here is an example of REST resource declaration:

~~~ java
{% include_snippet REST resource example ./comsat-jersey-server/src/test/java/co/paralleluniverse/fibers/jersey/TestResource.java %}
~~~

And then initiation of the jersey container:

~~~ java
{% include_snippet jersey registration ./comsat-jersey-server/src/test/java/co/paralleluniverse/fibers/jersey/FiberServletContainerTest.java %}
~~~
To learn about writing REST services with JAX-RS, please refer to the [Jersey User Guide](https://jersey.java.net/documentation/latest/user-guide.html).

{:.alert .alert-info}
**Note**: [Web Actors](webactors.html) are a great way to write REST services, as well as web-socket services, for interactive web applications.

### HTTP Clients

#### Apache Http Client
The fiber blocking version of the Apache Http Client can be used with FiberHttpClientBuilder.
For example:

~~~ java
{% include_snippet client configuration ./comsat-httpclient/src/test/java/co/paralleluniverse/fibers/httpclient/FiberHttpClientBuilderTest.java %}
~~~
After that you may call the refular API from fiber context:

~~~ java
{% include_snippet http call ./comsat-httpclient/src/test/java/co/paralleluniverse/fibers/httpclient/FiberHttpClientBuilderTest.java %}
~~~

If you prefer to use the future API of apacheHttpClient you should build regular HttpAsyncClient and then wrap it with FiberCloseableHttpAsyncClient.wrap, for example

~~~ java
{% include_snippet client configuration ./comsat-httpclient/src/test/java/co/paralleluniverse/fibers/httpasyncclient/FiberHttpAsyncClientTest.java %}
~~~

Then you can use it as follows:

~~~ java
{% include_snippet future calls ./comsat-httpclient/src/test/java/co/paralleluniverse/fibers/httpasyncclient/FiberHttpAsyncClientTest.java %}
~~~

#### Jersey Http Client
Comsat's integrated HTTP client is a JAX-RS client (specifically, Jersey client). To create a client instance compatible with Quasar fibers, use the [`AsyncClientBuilder`]({{javadoc}}/fibers/ws/rs/client/AsyncClientBuilder.html) class:

~~~ java
Client client = AsyncClientBuilder.newClient();
~~~

You can also pass a configuration

~~~ java
Client client = AsyncClientBuilder.newClient(config);
~~~

or use the builder API

~~~ java
{% include_snippet client creation ./comsat-jax-rs-client/src/test/java/co/paralleluniverse/fibers/ws/rs/client/AsyncClientBuilderTest.java %}
~~~

Then the usage is in the regular API, for example:

~~~ java
{% include_snippet http call ./comsat-jax-rs-client/src/test/java/co/paralleluniverse/fibers/ws/rs/client/AsyncClientBuilderTest.java %}
~~~

To learn how to use the HTTP client, please refer to the [Jersey documentation](https://jersey.java.net/documentation/latest/user-guide.html#client), or the [JAX-RS client Javadoc](http://docs.oracle.com/javaee/7/api/javax/ws/rs/client/package-summary.html).

All of the JAX-RS API is supported, and blocking calls are fiber- rather than thread-blocking. If you want to execute several requests in parallel, you may use any of the "async" methods that return a `Future`:

~~~ java
Future response = resourceTarget.request("text/plain").header("Foo", "bar").async().get(String.class);
~~~

Calling `Future.get()` would also just block the fiber and not any OS thread.

{:.alert .alert-info}
**Note**: A method that makes use of the API and runs in a fiber must be declared [suspendable](http://puniverse.github.io/quasar/manual/core.html#fibers) (normally by declaring `throws SuspendExecution`).

{:.alert .alert-warn}
**Note**: Jersey client's current implementation (since 2.5) has significant disadvantage relative to ApacheHttpClient since it uses thread per each open http call. Therefore it is not recommended, till it is fixed.

#### Retrofit

[`Retrofit`](http://square.github.io/retrofit/) lets you access REST api through java interface. In order to use it from fiber context you should first declare `Suspendable` interface:

~~~ java
{% include_snippet interface ./comsat-retrofit/src/test/java/co/paralleluniverse/fibers/retrofit/FiberRestAdaptherBuilderTest.java %}
~~~

This interface can be registered with `FiberRestAdaptherBuilder` and then used from fiber context.

~~~ java
{% include_snippet registration ./comsat-retrofit/src/test/java/co/paralleluniverse/fibers/retrofit/FiberRestAdaptherBuilderTest.java %}// ...
// usage from fiber context
{% include_snippet usage ./comsat-retrofit/src/test/java/co/paralleluniverse/fibers/retrofit/FiberRestAdaptherBuilderTest.java %}
~~~

### DB Access

#### JDBC

The `comsat-jdbc` project integrates the JDBC API with applications employing Quasar fibers (or actors). To use JDBC in Quasar fibers or actors, simply wrap your database driver's `DataSource` with [`FiberDataSource`]({{javadoc}}/fibers/jdbc/FiberDataSource.html), and use it to obtain connections which you may then freely use within fibers. For example:

~~~ java
{% include_snippet DataSource wrapping ./comsat-jdbc/src/test/java/co/paralleluniverse/fibers/jdbc/FiberDataSourceTest.java %}
~~~

Then the DataSource can be used with the regular API from fiber context, For example:

~~~ java
{% include_snippet DataSource usage ./comsat-jdbc/src/test/java/co/paralleluniverse/fibers/jdbc/FiberDataSourceTest.java %}// ...
// usage in fiber context
{% include_snippet connection usage ./comsat-jdbc/src/test/java/co/paralleluniverse/fibers/jdbc/FiberConnectionTest.java %}
~~~

{:.alert .alert-info}
**Note**: A method that makes use of the API and runs in a fiber must be declared [suspendable](http://puniverse.github.io/quasar/manual/core.html#fibers) (normally by declaring `throws SuspendExecution`).

Normally, Comsat transforms asynchronous (callback based) API into fiber-blocking operations. JDBC, however, has no asynchronous API. c`omsat-jdbc` simply runs the actual thread-blocking JDBC operations in a thread pool, and blocks the calling fiber until the operation has completed execution in the thread pool. As a result, you will not get any scalability benefits by calling your database in fibers (unlike, say, calling web services), because an OS thread will still block on every JDBC call. In practice, though, it matters little, as your database is likely to be a narrower bottleneck than the OS scheduler anyway.

{:.alert .alert-warn}
**Note**: Your application may only may direct use of the Comsat JDBC data source, because methods calling the API must be declared suspendable (or run on regular threads). Database access frameworks (like various ORM solutions) that make use of JDBC cannot use this data source and be used in Quasar fibers. In the future, we will provide separate integration module for some popular database access libraries.

If you want to learn how to use JDBC, the [JDBC tutorial](http://docs.oracle.com/javase/tutorial/jdbc/basics/) is a good resource.

#### JDBC Deployment Via JNDI

Servlets often make use of JDBC data sources exposed through JNDI. If you do that, you can declare a COMSAT (i.e. a fiber-aware) JDBC data source through JNDI that will wrap your native data source. To do so, you will use the `co.paralleluniverse.fibers.jdbc.FiberDataSourceFactory` DataSource factory, and pass in the number of threads you'd like COMSAT to use in the JDBC worker pool.

In order to do that first you have to include `comsat-jdbc-{{site.version}}.jar` in your container's runtime classpath, by putting it into the container's `lib` directory.

If you're using `TOMCAT`, the following example is a snippet of `META-INF/context.xml` that will declare a DataSource under the `jdbc/fiberdb` name, which wraps a native DB declared under the `jdbc/globalds` name:

~~~ xml
<Context path="/">
...
{% include_snippet fiber ds ./comsat-test-war/src/main/webapp/META-INF/context.xml %}</Context>
~~~

In order to do the same thing with `Jetty`, you have to include similar definition in your `WEB-INF/jetty-env.xml`:

~~~ xml
{% include_snippet fiber ds ./comsat-test-war/src/main/webapp/WEB-INF/jetty-env.xml %}
~~~

#### JDBI

To use the powerful API of [JDBI](http://jdbi.org/) to access databases you first have to create IDBI instance using the FiberDBI implementation:

~~~ java
{% include_snippet creation ./comsat-jdbi/src/test/java/co/paralleluniverse/fibers/jdbi/FiberFluentAPITest.java %}
~~~
The created instance can be used both with the Fluent API as well as with the SqlObject API. First the fluent API example:

~~~ java
{% include_snippet usage ./comsat-jdbi/src/test/java/co/paralleluniverse/fibers/jdbi/FiberFluentAPITest.java %}
~~~

And now to the SqlObjectAPI. In this API you have to declare first a `Suspendable` interface. Here is an example:

~~~ java
{% include_snippet interface ./comsat-jdbi/src/test/java/co/paralleluniverse/fibers/jdbi/FiberSqlObjectAPITest.java %}
~~~

The interface now can be registered and used in the regular manner from fiber context:

~~~ java
{% include_snippet registration ./comsat-jdbi/src/test/java/co/paralleluniverse/fibers/jdbi/FiberSqlObjectAPITest.java %}//...
// usage in fiber context
{% include_snippet usage ./comsat-jdbi/src/test/java/co/paralleluniverse/fibers/jdbi/FiberSqlObjectAPITest.java %}
~~~

#### jOOQ

Another possibility accessing a relational databases is [JOOQ](http://www.jooq.org/).
In order to use jOOQ from fiber context, all you have to do is to provide connection originated from `FiberDataSource`, For example:

~~~ java
{% include_snippet creation ./comsat-jooq/src/test/java/co/paralleluniverse/fibers/jooq/JooqContextTest.java %}// ...
// mapper definition
{% include_snippet mapper ./comsat-jooq/src/test/java/co/paralleluniverse/fibers/jooq/JooqContextTest.java %}// ...
// usage in fiber context
{% include_snippet usage ./comsat-jooq/src/test/java/co/paralleluniverse/fibers/jooq/JooqContextTest.java %}
~~~

#### MongoDB

Comsat integrates with MongoDB, for a fiber-blocking Mongo access via the [allanbank API](http://www.allanbank.com/mongodb-async-driver/index.html).

This is how you get a fiber-freindly `MongoDatabase` instance:

~~~ java
MongoClient mongoClient = FiberMongoFactory.createClient( "mongodb://localhost:" + port + "/test?maxConnectionCount=10" ).asSerializedClient();
MongoDatabase mongoDb = mongoClient.getDatabase("mydb");
~~~

### Dropwizard

[Dropwizard](http://dropwizard.io/) is a Java framework for developing ops-friendly, high-performance, RESTful web services. You can find sample for dropwizard-comsat application [here](https://github.com/puniverse/comsat/blob/master/comsat-dropwizard/src/test/java/co/paralleluniverse/fibers/dropwizard/MyDropwizardApp.java).

In order to use dropwizaed in a Comsat environment, only few changes have to be made in the application declartion.
First the yaml configuration file:

~~~ yaml
{% include_snippet server ./comsat-dropwizard/src/test/resources/server.yml %}
~~~
The number of thread needed for comsat container is not high even if the number of concurrent connection is. You can leave it with 50 or 200 threads, but you should enlarge the queue size configuration in order to handle this amount of connection. You also have to configure apropriate requestLog appender or shut it down. Next is the httpClient configuration:

~~~ yaml
{% include_snippet httpclient ./comsat-dropwizard/src/test/resources/server.yml %}
~~~
Here also you should enlarge `maxConnections` properties to support many connections at the same time. The db configuration should be looked like this:

~~~ yaml
{% include_snippet db ./comsat-dropwizard/src/test/resources/server.yml %}
~~~

The `driverClass` has to point to the `co.paralleluniverse.fibers.jdbc.FiberDriver` class. The `url` has to be the url of your real datasource with the addition of 'fiber:' prefix, and as ussual you should include the driver of the your real datasource in your runtime classpath.

Now we can move to the code declaratations:

~~~ java
{% include_snippet app ./comsat-dropwizard/src/test/java/co/paralleluniverse/fibers/dropwizard/MyDropwizardApp.java %}
~~~
Instead of extending the regular `io.dropwizard.Application` class, you should extend the Comsat's `FiberApplication`. Your regular `run` function should be slightly changed to be called `fiberRun`. The creation of httpClient should be done using the `FiberHttpClientBuilder` class and the creation of `jdbi` should be done using the `FiberDBIFactory` class.

## Web Actors

Web Acotrs are [Quasar actors](http://puniverse.github.io/quasar/manual/actors.html) that receive and respond to messages from web clients. Web actors support HTTP, SSE and SSE (Server-Sent Events) messages, and are a convenient, efficient, and natural method for implementing the backend for interactive web applications.

### Deployment

WebActors are implemented on top of a web server. Currently, they can be deployed be deployed in any JavaEE 7 servlet container, but we are working on supporting deploying them on top of [Netty](http://netty.io/) and [Undertow](http://undertow.io/).

A web actor is attached to a web session. It can be spawned and attached manually (say, after the user logs in and the session is authenticated). The manual attachment API is container dependent: see [here](??????) for the API for JavaEE containers. A web actor can also be spawned and attached automatically by letting COMSAT spawn and attach a web actor to every newly created session. This method will be described below. Because a web actor consumes very few resources, spawning them automatically is sufficient in all but the most extreme circumstances.

For automatic deployment, all you have to do is define an actor class (one that extends `BasicActor` or `Actor`), and annotate it with the [`WebActor`]({{javadoc}}comsat/webactors/WebActor.html) annotation. For example:

~~~ java
@WebActor(name="chat", httpUrlPatterns="/chat", webSocketUrlPatterns="/chat/ws")
public class ChatActor extends BasicActor<WebMessage, Void> {
    @Override
    protected Void doRun() {
        // ...
    }
}
~~~


In this example, all HTTP requests to the `/chat` resource, as well as all websocket messages to `/chat/ws` will be received as messages by the actor. A new `ChatActor` will be spawned for every new HTTP session.

#### Embedded containers

If you use embedded container, you have to register `WebActorInitializer` as a ServletContextListener to your servletContainer. It will scan and register the webactors according to the @Webactor annotations. This registration should be looked somethink like this:

~~~ java
{% include_snippet WebActorInitializer ./comsat-actors-servlet/src/test/java/co/paralleluniverse/comsat/webactors/servlet/WebActorServletTest.java %}
~~~

Webactors may use websockets. In order to do that the container has to be configured to support it. Each container has its own way. In Jetty you have to include the `javax-websocket-server-impl` jar and call the following method before you start the container:

~~~ java
WebSocketServerContainerInitializer.configureContext(context);
~~~

In tomcat you have to include the `tomcat-embed-websocket` jar and register `ServletContainerInitilizer`:

~~~ java
context.addServletContainerInitializer(new WsSci(), null);
~~~


For details, see the [Javadoc]({{javadoc}}/comsat/webactors/WebActor.html).

### Basic Operation

A web actor should be able to receive messages of type [`WebMessage`]({{javadoc}}/comsat/webactors/WebMessage.html). `WebMessage` is the supertype of all messages that can be received from or sent to a web client. The class encapsulates a message body, that can be either text or binary, and a *sender*, which, following a common actor pattern, is the actor that sent the message.

For messages received from the web client, the *sender* is a virtual actor representing the web client. You can perform normal actor operations on it, like `watch` to detect actor death; their semantics depend on the specific type of the message.

A single web actor instance may handle HTTP requests, emit SSE events, and handle one or more WebSocket connections.

### HTTP (REST Services)

A web actor is attached to one or more HTTP resources (as specified by `@WebActor`'s `httpUrlPatterns` property), and an actor instance is associated with a single HTTP session. Every HTTP request to the resource, associated with the session, will be received by the actor as an [`HttpRequest`]({{javadoc}}/comsat/webactors/HttpRequest.html) message. The actor can then respond with an [`HttpResponse`]({{javadoc}}/comsat/webactors/HttpResponse.html) message, which it sends to the request's sender.

All HTTP request messages to a specific web actor instance will come from the same sender. If you `watch` that sender actor, it will emit an `ExitMessage` (signifying its death), when the session is terminated.

When you responsd to an `HttpRequest` with an `HttpResponse`, by default, the request stream will close. If, however, you wish to send the response's body in parts (e.g., for SSE, discussed in the next section), you may call `HttpRequest.openChannel`, which will return a Quasar *channel* that can be used to send [`WebDataMessage`]({{javadoc}}comsat/webactors/WebDataMessage.html)s messages to the stream. The stream will flush after each `WebDataMessage`'s body has been written to it. If `openChannel` has been called, the HTTP response stream will be closed when `close` is called on the returned channel.

Please refer to the [Javadoc]({{javadoc}}/comsat/webactors/HttpRequest.html) for details.

### SSE

SSE, or [Server-Sent Events](http://dev.w3.org/html5/eventsource/), is an HTML5 stnadard, supported by most modern browsers, for pushing discrete messages to the web client, without it sending new HTTP requests for each one. A good tutorial by Eric Bidelman on SSE can be found [here](http://www.html5rocks.com/en/tutorials/eventsource/basics/). An SSE stream is initiated with an HTTP request; then, each event message is written to the response stream and flushed, only the messages need to be encoded according to the SSE standard. SSE also specifies that if the connection is closed, the client will attempt to reconnect with a new request after a timeout, that can be set by the server.

The [`SSE`]({{javadoc}}/comsat/webactors/SSE.html) class contains a set of static utility methods that encode the events according to the SSE standard, and ensure that the response headers are set correctly (in terms of character encoding, etc.).

To start an SSE stream in response to an `HttpRequest`, do the following:

~~~ java
request.getFrom().send(new HttpResponse(self(), SSE.startSSE(request)));
~~~

This will set `HttpResponse`'s `startActor` flag, which will leave the response stream open and send back an [`HttpStreamOpened`]({{javadoc}}/comsat/webactors/HttpStreamOpened.html) message from a newly created actor representing the SSE stream. Once you receive
the message, you send SSE events by sending `WebDataMessage`s to that actor:

~~~ java
sseActor.send(new WebDataMessage(self(), SSE.event("this is an SSE event!")));
~~~

To close the stream, you send a `co.paralleluniverse.actors.ShutdownMessage` to the SSE actor like so:

~~~ java
co.paralleluniverse.actors.ActorUtil.sendOrInterrupt(sseActor, new ShutdownMessage());
~~~

It might be convient (and elegant) to wrap the channel returned by `openStream` with a *mapping channel* (see the Quasar docs), that will transform a message class representing the event into an SSE-encoded `WebDataMessage`.

### WebSockets

[WebSocket](http://en.wikipedia.org/wiki/WebSocket) is a new web protocol for low(ish)-latency, bi-directional communication between the client and the server. Web sockets are extremely useful for interactive web applications, and they fit beautifully with COMSAT Web Actors.

A web actor may register itself to handle web socket connections by declaring which WebSocket URIs it is interesten in, in the `@WebActor` annotations `webSocketUrlPatterns` property. Such a web actor will handle all web socket sessions at those URIs associated with the actor instance's HTTP session (a web socket is also associated with an HTTP session).

When the client connects to a web socket, the web actor will receive a [`WebSocketOpened`]({{javadoc}}/comsat/webactors/WebSocketOpened.html) message, and each following message will be received as a [`WebDataMessage`]({{javadoc}}comsat/webactors/WebDataMessage.html). The actor can send messages to the client by replying to the sender with `WebDataMessage`s of its own.

The virtual actor that's the *sender* of the messages received from the client represents the WebSocket session; i.e., each open web socket will have a different actor as the sender of the messages. That virtual actor dies when the web socket connection closes.

## Examples

This GitHub project contains examples covering most of the COMSAT functionality: [puniverse/comsat-examples](https://github.com/puniverse/comsat-examples).
