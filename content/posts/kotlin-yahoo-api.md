---
title: "Yahoo Kotlin API on AWS"
description: ""
date: 2021-11-05T13:16:16+01:00
lastmod: 2021-11-05T13:16:16+01:00
draft: false
author: "Auryn Engel"

tags: ["development", "aws", "kotlin", "HowTo"]
categories: ["development"]

---

## Getting started

I have created a node lib, to load stock prices from yahoo finance and calculate your current net worth. But here I load all these data directly and in every iteration myself.

To reduce the load to yahoo and create an API for friends, I have decided to create an API with Kotlin and Micronaut. To learn Micronaut in more detail.
And to document everything, I started this blog post here.

So let's get started.

First of all, we need to start a Kotlin project. Normally I work with maven, but let's check out gradle. The start is documented [here](https://docs.gradle.org/current/samples/sample_building_kotlin_applications.html). But I will also document my steps here in more detail.

```bash
gradle init
```

Then we have to select everything like this:

```txt
Welcome to Gradle 7.2!

Here are the highlights of this release:
 - Toolchain support for Scala
 - More cache hits when Java source files have platform-specific line endings
 - More resilient remote HTTP build cache behavior

For more details see https://docs.gradle.org/7.2/release-notes.html

Starting a Gradle Daemon (subsequent builds will be faster)

Select type of project to generate:
  1: basic
  2: application
  3: library
  4: Gradle plugin
Enter selection (default: basic) [1..4] 2

Select implementation language:
  1: C++
  2: Groovy
  3: Java
  4: Kotlin
  5: Scala
  6: Swift
Enter selection (default: Java) [1..6] 4

Split functionality across multiple subprojects?:
  1: no - only one application project
  2: yes - application and library projects
Enter selection (default: no - only one application project) [1..2] 1

Select build script DSL:
  1: Groovy
  2: Kotlin
Enter selection (default: Kotlin) [1..2] 1

Project name (default: yahoo-micronaut):
Source package (default: yahoo-micronaut):

> Task :init
Get more help with your project: https://docs.gradle.org/7.2/samples/sample_building_kotlin_applications.html

BUILD SUCCESSFUL in 43s
2 actionable tasks: 2 executed
```

The basic structure will look like this:

```txt
.
├── app
│   ├── build
│   │   ├── classes
│   │   │   └── kotlin
│   │   │       └── main
│   │   │           ├── META-INF
│   │   │           │   └── app.kotlin_module
│   │   │           └── yahoo
│   │   │               └── micronaut
│   │   │                   ├── App.class
│   │   │                   └── AppKt.class
│   │   └── ...
│   ├── build.gradle
│   └── src
│       ├── main
│       │   ├── kotlin
│       │   │   └── yahoo
│       │   │       └── micronaut
│       │   │           └── App.kt
│       │   └── resources
│       └── test
│           ├── kotlin
│           │   └── yahoo
│           │       └── micronaut
│           │           └── AppTest.kt
│           └── resources
├── build
│   └── kotlin
│       └── sessions
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
└── settings.gradle

14 directories, 8 files
```

To run this code, just type `gradle run` and the response should be this:

```txt
> Task :app:run
Hello World!

BUILD SUCCESSFUL in 705ms
2 actionable tasks: 1 executed, 1 up-to-date
```

The command `gradle test` should run all tests successfully.

So, let's start with the actual code.

## Build the code

To build the project we the [Gradle Shadow plugin](https://ktor.io/docs/fatjar.html) from Ktor.
Ther you only have to configure the plugin as described [here](https://ktor.io/docs/fatjar.html) and run `gradle build` to get the fat jar to run with java (`java -jar app/build/libs/app-0.1-all.jar`).

## The Code

We start with the `App.kt`. Here we will start Micronaut and run the code.

```kotlin

// instatiate logger for usage
val logger = KotlinLogging.logger {}

fun main(args: Array<String>) {
    Micronaut.build()
            .args(*args)
            .packages("yahoo.micronaut")
            .start()
}
```

From there we the `RestController`, where the Endpoints are defined:

```kotlin
@Controller("/api")
class RestController(private val yahooController: YahooController) {

    @Get("/{ticker}")
    fun greet(ticker: String): Mono<StockPrice> {
        logger.info { "Request for ticker $ticker" }
        return Mono.justOrEmpty(yahooController.loadStockForTicker(ticker))
    }
}
```

We `injected` the YahooController to call the `API` from yahoo and return the `StockPrice`.
The `YahooController` is pretty simple. There we only call a function to get the data from yahoo and afterwards pipe this back to our `REST` endpoint. We are doing this for best practice and more importantly, to cache the request. The Yahoo Website is pretty slow. And if we want to have a fast endpoint for multiple callers, we don't want to call the Yahoo Website on every request for the same ticker. We want to call Yahoo just once a minute.

```kotlin
// this is only once instantiated and the config is in our application yaml 
@Singleton
@CacheConfig("stocks")
open class YahooController {

    // enable caching for this function
    // because we just have one parameter, we don't need a parameter for this annotation
    @Cacheable
    open fun loadStockForTicker(ticker: String): StockPrice? {
        return loadStockPrice(ticker)
    }
}
```

Now we just need the `YahooReader` where we have the functions to call the website and parse the data from it.

```kotlin
// This is the base URL where we will get the ticker data from
private const val BASE_URL = "https://finance.yahoo.com"


// load the data from the Yahoo url and return the StockPrice if found
fun loadStockPrice(ticker: String): StockPrice? {
    // log the request
    logger.info {"Load data for ticker: $ticker from $BASE_URL"}
    val client = HttpClient.newBuilder().build()
    // build the request
    val request = HttpRequest.newBuilder()
            .uri(URI.create("$BASE_URL/quote/$ticker/"))
            .build()
    // load the data and return the body as string
    val response = client.send(request, HttpResponse.BodyHandlers.ofString()).body()
    // parse the data and return the result
    return getStockPriceFromResponse(response, ticker)
}

// find the json in the website and return this part
fun findJsonStockPriceFromWebsite(response: String, ticker: String): String {
    val startPosition = response.indexOf("\"$ticker\":{\"sourceInterval\"") + "\"$ticker\":".length
    var index = startPosition
    var bracketsCounter = 0
    for (i in index..response.length) {
        if (response[i] == '{') {
            bracketsCounter++
        }
        if (response[i] == '}') {
            bracketsCounter--
        }
        if (bracketsCounter == 0) {
            index = i
            break
        }
    }
    return response.substring(startPosition, index + 1)
}

// try to parse the selected string from the website
fun getStockPriceFromResponse(response: String, ticker: String): StockPrice? {
    return try {
        StockPrice.fromJson(findJsonStockPriceFromWebsite(response, ticker))
    } catch (e: KlaxonException) {
        logger.info { "Could not parse response for ticker: $ticker" }
        null
    }
}
```

In the `model` we added the `StockPrice` model. This is created from the `JSON` we receive from the call with the [JSON Formatter](https://jsonformatter.org/json-to-kotlin).

## Deploy the Code

The last step is to deploy our code to a cloud provider. I will use AWS and go through [this](https://guides.micronaut.io/latest/micronaut-elasticbeanstalk-gradle-kotlin.html) tutorial from micronaut.

![AWS Config](/img/micronaut/aws-env.png)

Important is, to set micronaut to the port 5000, so aws can recognize the port for the health check. Do this by adding an environment variable (`MICRONAUT_SERVER_PORT`) to `5000`.

## Conclusion

And thats it. This is how you build your own super slim microserver to host an API on AWS. Here we created an endpoint to host yahoo stock data and cache them for one minute. So the initial response is slow, but the second response in this minute is blasting fast 🚀.

Have fun and create some awesome API's with kotlin and micronaut.

You can find the code on my [GitHub](https://github.com/auryn31/yahoo-kotlin-api).

## Extra information

### Set Java to version 11

- install [sdk](https://sdkman.io/install)
- `sdk install java 11.0.2-open`
- `sdk use java 11.0.2-open`
- `java --version`

```txt
    openjdk 11.0.2 2019-01-15
    OpenJDK Runtime Environment 18.9 (build 11.0.2+9)
    OpenJDK 64-Bit Server VM 18.9 (build 11.0.2+9, mixed mode)
```

### Dependencies

I had a lot of trouble adding the correct dependencies. Next time i just will use `mn create-app <NAME> --build gradle_kotlin`. Here you have all the correct dependencies to start. Here you have my `build.gradle`.
