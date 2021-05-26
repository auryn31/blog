---
title: "Comparison of Microservice Frameworks with a Streaming Example"
subtitle: ""
date: 2019-07-08T21:10:50+02:00
lastmod: 2021-05-26T21:10:50+02:00
draft: false
author: "Auryn Engel"
authorLink: "/about"

tags: []
categories: []

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

In my last [post](/posts/reactive-stream-with-spring-boot-and-rx-java-in-kotlin/), I presented some interesting applications of reactive programming with RxJava and when/why you should use it.
<!--more-->

These services could be ramped up or down depending on the load. If a service is heavily loaded, a load balancer could now simply start additional services, redirect the load and thus relieve one and reduce the response time again. If the services are under little load, they could be stopped, thus freeing up performance for other services or saving costs if a cloud service is used.

Services are rarely free of load peaks, here it is advantageous if the starting and stopping of further units is fast and these newly started instances react quickly to requests in order to keep the response times for all requests low.

Therefore, in this article I will look at various Microservice frameworks and compare them in terms of start time, time to first response and response time under load. In addition, I will briefly discuss the simplicity of getting started with the frameworks.

In the first step I will deal with the following frameworks/servers (*Micronaut, Wildfly, Dropwizard* and *Spark* will follow in due course):

- Spring Boot
- Vert.x
- Helidon
- Quarkus

I started with *Spring Boot*, because I already used it in my first article and developed it in production.

The following post is based on the previous post on *Reactive Stream with Spring Boot*. Here I have already created a stream REST endpoint and developed a small frontend for illustration. The code for the following backends will be available in the same [GitHub-Repo](https://github.com/auryn31/spring-async-rest-example).

Excitingly, *Spring Boot* has the default for `application/stream+json`. However, this is not a MIME type. For me this is unclear, because the format seems to make sense.

Since *Quarkus* and *Helidon* use JAX-RS (which uses the *MIME* standard), either the type itself must be defined or an `application/octet-stream` must be used.

So that I don’t have to change the frontend, I used both the `application/octet-stream` and `application/stream+json`. The respective endpoints are accessible via headers. Without an header a`pplication/json` is returned.

## Entry

In the following section I will briefly look at the frameworks. I myself only used *Spring Boot* in production before, so the other frameworks are also new territory for me. I will briefly explain the difficulties I had in creating the corresponding endpoint in the frameworks.

### Spring Boot

*Spring Boot* is probably the best known of the frameworks. The *>38.000* stars on Github speak for themselves.

Getting started into *Spring Boot* is kept very simple by . Here a **pom.xml** with all necessary dependencies is created and you can start immediately. In addition, the community is very large and you can find instructions on how to handle most problems.

Spring Boot was the only framework that supported `application/stream+json` by default. It also offers many other features and a lot of help for a clear and easy development.

Since I use *Spring Boot* a lot myself, it was easy for me to create the appropriate endpoints. As you can see below, it doesn’t take much to create a *Response Stream*:

```kotlin
@Controller
class RestEndpoint {

    @Autowired
    lateinit var dataProvider: DataProvider

    @Autowired
    lateinit var streamResponse: CarStreamResponseOutput

    @GetMapping(path = ["cars"], produces = [MediaType.APPLICATION_STREAM_JSON_VALUE])
    fun getCarsAsStream(): StreamingResponseBody {
        return streamResponse
    }

    @GetMapping(path = ["cars"], produces = [MediaType.APPLICATION_JSON_VALUE])
    fun getCarsAsJson(): List<Car> {
        return dataProvider.getDataStream().toList().blockingGet()
    }
}
```

For the stream response, however, a *StreamingResponseBody* is required:

```kotlin
@Component
class CarStreamResponseOutput : StreamingResponseBody {
    @Autowired
    lateinit var dataProvider: DataProvider

    override fun writeTo(os: OutputStream) {
        val writer = BufferedWriter(OutputStreamWriter(os))
        val countDownLatch = CountDownLatch(1)
        dataProvider.getDataStream().subscribe({
            writer.write(Klaxon().toJsonString(it))
            writer.write("\n")
            writer.flush()
        }, ::println, {
            os.close()
            countDownLatch.countDown()
        })
        countDownLatch.await()
        writer.flush()
    }
}
```

Basically thats it. So it goes to the next framework.

### Vert.x

*Vert.x* has quite good documentation. It was developed by the *Eclipse Foundation* and has been designed directly for reactive applications on the JVM. Nevertheless, I couldn’t just pass my observable (or flowable) to the response handler. Theoretically you can return a flowable directly as described in the documentation, but the flowable does not write directly to the stream at every new event. It seems to buffer the elements in the flowable and only write the stream when the event done comes from the flowable. But to get a continuous stream you have to write your own handler, which turned out to be not very complex. This is very similar to *Spring Boot*.

```kotlin

override fun handle(rtx: RoutingContext) {
  val response = rtx.response()
  response.setChunked(true)
  val flow: Flowable<String> = DataService.getDataStream(TIMEOUT).map { Klaxon().toJsonString(it) }.toFlowable(BackpressureStrategy.BUFFER)
  flow.subscribe({
    response.write(it)
    response.write("\n")
    response.writeContinue()
  }, ::println, {
      response.end()
  })
}
```

Also otherwise the documentation of *Vert.x* is good and the community with more than *9700* GitHub Stars is constantly growing.

The application is compiled by `./mvnw clean compile`, started by `./mvnw exec:java`. The commands can also be easily combined. (`./mvnw clean compile exec:java`)

All in all you find yourself in *Vert.x* and can start developing quickly. But you have to get used to developing on a main thread, because you can’t block it. In the beginning I had the error that I used *Thread.sleep* and therefore the performance was very limited. But this is also described as *DON'T* on the website. After this was fixed, *Vert.x* could score again with performance. The other frameworks got along with it, despite it, it was also taken out of them, because it was in the *domain* part of the application, which all frameworks share.

### Helidon

*Helidon* sets to the JAX-RS standard like *Quarkus*. So I could use the same code as for *Quarkus*. You only have to register a *JerseySupport* and off you go. The positive thing about *Helidon* is that it doesn’t need any own commands in the terminal to be started. Here the IDE support is very simple and pleasant, since all necessary dependencies come with the pom. So building can be done with a simple mvn clean install and the built jar can be done with `java -jar`. It has to be said that `mvn clean install` is also enough for all other frameworks to build an executable `jar`. *Vert.x* and *Quarkus* only bring more scripts and need an extra class to be started from the IDE. This extra class doesn't come with both by *default*.

```kotlin
fun main(args: Array<String>) {

    val serverConfig = ServerConfiguration.builder()
        .port(8080).build()

    val webServer = WebServer
        .create(serverConfig, Routing.builder()
                .register("/cars", JerseySupport.builder().register(CarService::class.java).build())
                .build())
        .start()
        .toCompletableFuture()
        .get(10, TimeUnit.SECONDS)
}
```

### Quarkus

*Quarkus* is still a quite young framework, it already gets a lot of attention in the community. It is currently at nearly *2000* stars on GitHub. It was and is developed by **Red Hat**. Nativly it compiles for the GraalVM, but can also be compiled for the classic JVM. But here it doesn’t show its strengths by the slim RAM consumption and the extremely fast starttime. Although, as we’ll see later, it still starts on the JVM under one second. It uses various standards, including JAX-RS Netty and Eclipse MicroProfiles.

*Quarkus* writes itself a very fast start and therefore scaling and a low memory consumption on the flag. In addition, the manufacturers also rely on the reactive approach to develop highly concurrent and responsive applications. For this there is a more detailed article in the JavaSpektrum (7/2019), in which Quarkus is examined more exactly. Among other things, it shows that the application on the JVM consumes *100mb* RAM, whereas on the GraalVM it needs only *8mb*.

The documentation of *Quarkus* is detailed and easy to read. Unfortunately, there are only a few tutorials and explanations so far, because the community is not so big yet and the framework has not been in use long enough. If problems occur, you have to search for a long time or ask your own questions to the community. But just because of the fact that *Quarkus* comes from **Red Hat**, the community won’t be long in coming.

The advantage is that you can develop a fast and lean application with the existing *Java* or *Kotlin* knowledge.

```kotlin
@Path("/cars")
class CarResource {
    @Inject
    lateinit var responseStream: CarStreamResponseOutput

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    fun getCarsAsList() = DataService.getDataStream(0).toList().blockingGet()

    @GET
    @Produces("application/stream+json")
    fun loadCarsAsJsonStream(): Response {
        return Response.ok().entity(responseStream).build()
    }
}
```

## Result

In the following section I compare and evaluate the different frameworks.

### Development

*Spring Boot*, *Quarkus* and *Helidon* use almost the same `ResponseWriter`, *Vert.x* uses a handler. In *Helidon* and *Quarkus* you can rely on the classic JAX-RS approach. Due to the *Java EE* development there is a lot of documentation here. With *Vert.x* there is however a good own documentation. All in all, developing in *Spring Boot* was easiest for me. This is partly due to the experience, partly due to the currently largest community. However, the advantages of the other frameworks cannot be denied, as we will see from the numbers.

### Tests of the different backends

In the first step all backends on the JVM (*java version "11.0.2"*) are started. Then the corresponding endpoints are addressed with curl. The first-response times are determined by a format file (located in my GitHub repo).

```bash
curl -w "@curl-format.txt" -o /dev/null -s "http://localhost:8080/cars" -H "Accept:application/stream+json"
```

*Average* and *Median* response times are determined with `k6`. Here 10 simulated users for *30s* requests are sent to the point. The results can be found in the following table.

|Criterion      |Spring Boot|Vert.x|Helidon|Quarkus|
|---------------|-----------|------|-------|-------|
|Starttime      |2.226s     |0.200s|0.619s |0.562s |
|First Response |0.190s     |0.350s|0.540s |0.523s |
|RPS Small      |8712       |5372  |7082   |9269   |
|RPS Large      |79         |99    |98     |98     |
|Average Response Time small Data|1.12ms|1.32ms|1.39ms|1.05ms|
|Median Resonse Time small Data|1.03ms|1.7ms|1.13ms|0.914ms|
|Average Response Time large Data|126ms|101ms|102ms|101ms|
|Median Response Time large Data|118ms|101ms|101ms|101ms|

The start time of the four frameworks is shown in the following picture. Here you can see that *Spring Boot* is clearly beaten by the other frameworks.

![starttime](/img/comparison-of-microservice-frameworks/starttime.jpeg "Starttime")

The slow start time only has the advantage that more dependencies are loaded and the first response to a query is faster. This can be seen below.

![first response](/img/comparison-of-microservice-frameworks/first-response.jpeg "First Response")

In the following two pictures, the responses are displayed per second. *Small* are data without delay, *large* are data with a delay of *100ms*. As can be seen here, the different frameworks are relatively similar with delay, whereby *Spring Boot* was about 20% slower. However, if the response is fast and small, the differences are greater. Here Vert.x is almost 50% slower than *Quarkus*.

![small response](/img/comparison-of-microservice-frameworks/small-response.jpeg "Small Response")
![large response](/img/comparison-of-microservice-frameworks/large-response.jpeg "Large Response")

### Load tests with wrk

I repeated the tests with the [wrk](https://github.com/wg/wrk). The tool is very popular for *http* load tests. The tests were performed as above on the stream endpoint.

```bash
wrk -c 400 -d 10 - latency - timeout 1s http://localhost:8080/cars-locust
```

|Criterion      |Spring Boot|Vert.x|Helidon|Quarkus|
|---------------|-----------|------|-------|-------|
|RPS Small      |10020      |6096|7663 |15070|
|RPS Large      |30         |3346|927 |930 |

As you can already see in the table, the results for the quick answer are similar to those with *k6*. Only Quarkus was able to handle almost twice as many requests at wrk. With slow data, Quarkus could act again most answers, but also gave most *5xx* answers. Here only the *2xx* answers were used. There *Vert.x* could stand out clearly before the others. This can also be seen very well in the following diagram:

![rps wrk](/img/comparison-of-microservice-frameworks/rps-wrk.jpeg "rps wrk")

If *Vert.x* is started as a single instance, it cannot handle as many requests with fast responses. But since *Vert.x* is intended to be started with multiple instances, I tested it again with 8 instances. It was able to answer almost as many questions as Quarkus.

![rps wrk small](/img/comparison-of-microservice-frameworks/rps-wrk-small.jpeg "rps wrk small")

## Conclusion

Different tools, different results. Under load the answers for a quick answer were similar for both tools, do they differed greatly for slow answers. In the first response, *Spring Boot* was well ahead, but it loses a lot of start time. At the start time, no framwork could beat *Vert.x*. *Vert.x* could also show at *wrk* what it is able to do. Here it could parralel process almost 100 times more answers than *Spring Boot*. And that despite only one instance. Spring could beat the simple instance of *Vert.x* for a small answer, but the picture changes if you start multiple instances of *Vert.x*.

All in all, *Vert.x* was the most convincing in the picture. It’s more stable than Quarkus, faster than Helidon and *Spring Boot* and has very good documentation. With Quarkus I often got into invalid states and had to restart it. This can be avoided by better error handling. If it is about the amount of documentation, help in the net and developers, you should probably set to *Spring Boot*. If you want to start and stop the service quickly, it’s worth the time to invest and use a new service like Quarkus or *Vert.x*. This is especially exciting when using a microservice architecture.

Personally I will try to use *Vert.x* in the next project and concentrate on it in the future. Especially in combination with *Kotlin* a very exciting topic.

The complete code can be found on my [GitHub-Repo](https://github.com/auryn31/spring-async-rest-example).

*Note:* this article is imported from [medium](https://medium.com/swlh/comparison-of-microservice-frameworks-with-a-streaming-example-6bfe284a66a) where i previously published my articles.
