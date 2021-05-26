---
title: "HowTo: Reactive Stream with Spring Boot and RxJava in Kotlin"
date: 2019-05-18T19:24:01+02:00
lastmod: 2021-05-26T19:24:01+02:00
draft: false
author: "Auryn Engel"

tags: ["HowTo", "Kotlin", "Spring Boot", "RxJava"]
categories: ["Development"]

---

Everyone knows that, you open a website and load and load it. 😣

But why is that?

<!--more-->
Data is loaded from the server and must be displayed on the website. If this data is not available quickly, this can lead to long loading times because the server has to prepare the data first. But the server can actually make some of the data available as soon as it is available, not when all the data has been collected together, right?👌

This can be achieved, for example, by streams. So an asynchronous loading of the data from the server to the client (website). The data is sent to the client as soon as a part is available. 👨‍💻

In the following we will see how we can implement this with **RxJava**, **Kotlin** and **Spring Boot**. Furthermore, we will create a synchronous endpoint and a Vue.js page to illustrate the difference.

## Lets start

First of all we need data we want to display. Let’s take cars:

```kotlin
data class Car(val name: String, val model: String, val company: String)
```

Now we have created a model in Kotlin, which we can provide asynchronously with RxJava. In the following we will create an observable, which can forward every incoming object to the client. How observables work is described on many other pages and goes a bit too far. However, on the [RxMarbles](https://rxmarbles.com/) page you can see very well how different functions work on an observable.

```kotlin
fun getDataStream() : Observable<Car> {
    val publishSubject = PublishSubject.create<Car>()
    GlobalScope.async {
        Thread.sleep(100)
        emitRandomCarsWithTimeout(publishSubject)
    }
    return publishSubject
}

fun emitRandomCarsWithTimeout(publishSubject: PublishSubject<Car>) {
    for (i in 0..10) {
        Thread.sleep(150)
        publishSubject.onNext(createRandomCar())
    }
    publishSubject.onComplete()
}
```

The *createRandomCar()* function creates a new random car. This can be seen on the [GitHub-Repo](https://github.com/auryn31/spring-async-rest-example).

A timeout of 150ms is generated between each car, for example to simulate a slow connection to the database or loading other REST services.

The data generated in this way can be made available as a REST service:

```kotlin
@Controller
class RestEndpoint {

    @Autowired
    lateinit var dataProvider: DataProvider

    @GetMapping(path = ["cars"], produces = [MediaType.APPLICATION_STREAM_JSON_VALUE])
    @ResponseBody
    @ApiResponses(value = [ApiResponse(code = 200, message = "Cars")])
    fun getCarsAsStream(): Observable<Car> {
        return dataProvider.getDataStream()
    }

    @GetMapping(path = ["cars"], produces = [MediaType.APPLICATION_JSON_VALUE])
    @ResponseBody
    @CrossOrigin(origins = ["http://localhost:8081"])
    fun getCarsAsJson(): List<Car> {
        return dataProvider.getDataStream().toList().blockingGet()
    }

}
```

We set the *CrossOrigin* header, because we want to request the data from a local Vue.js application later on and we don’t get it otherwise.

Now two endpoints have been created which send back a stream or a list of cars depending on the header.

If we now load this data asynchronously, we get the following answer:

```json
{"id":44095,"model":"Audi","company":"e-tron"}
{"id":8272,"model":"Ford","company":"Kuga"}
{"id":63213,"model":"Kia","company":"Opa"}
{"id":41440,"model":"Fiat","company":"Kuga"}
{"id":33670,"model":"Toyota","company":"Kuga"}
{"id":66710,"model":"Ford","company":"Nova"}
{"id":64250,"model":"VW","company":"Opa"}
{"id":83594,"model":"Chevrolet","company":"iMIEV"}
{"id":70848,"model":"Audi","company":"TT Coupé"}
{"id":55812,"model":"Chevrolet","company":"iMIEV"}
{"id":81105,"model":"Audi","company":"Kuga"}
```

This data can also be loaded synchronously, then it looks like this:

```json
[
  {"id":36599,"model":"Audi","company":"Probe"},
  {"id":30709,"model":"Kia","company":"Probe"},
  {"id":62511,"model":"Kia","company":"Phaeton"},
  {"id":95672,"model":"Fiat","company":"Pinto"},
  {"id":19564,"model":"Mercedes","company":"Pinto"},
  {"id":88003,"model":"VW","company":"Uno"},
  {"id":72413,"model":"Mercedes","company":"Phaeton"},
  {"id":18516,"model":"Fiat","company":"e-tron"},
  {"id":21171,"model":"Ford","company":"Opa"},
  {"id":27514,"model":"VW","company":"Uno"},
  {"id":21767,"model":"Mercedes","company":"Nova"}
]
```

You can see the difference in loading very well if you use it in a website.

For asynchronous loading, we use oboe.js in the Vue.js application, for synchronous loading we use axios. Both are well known libraries for sending requests to servers.

Best recognizable is the result in the GIF on the [Github-Repo](https://github.com/auryn31/spring-async-rest-example).

Otherwise try it yourself, download the repo and test the difference in the UX.

## Summary

Creating an Asynchronous Residual Endpoint is not difficult, but the difference in the UX is significant. So reactive programming should be used as much as possible to give the user a particularly good behavior of the application. After all, what does a user do if he doesn’t happen after clicking a button? Exactly, click again. Thus the server is further loaded, the user does not get an answer and everyone is frustrated. So, use reactive programming! 👨‍💻

The code is available on my [Github-Repo](https://github.com/auryn31/spring-async-rest-example).

Leave me a comment. 👏