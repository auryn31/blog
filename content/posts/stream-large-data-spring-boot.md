---
title: "HowTo: Stream large data with Spring Boot and RxJava in Kotlin"
date: 2019-07-10T17:40:18+02:00
lastmod: 2021-05-26T17:40:18+02:00
draft: false
author: "Auryn Engel"

tags: ["HowTo", "Kotlin", "Spring Boot"]
categories: ["Development"]

---
Imagine you have a very large amount of data that you want to make available at a rest-endpoint. Then there are two possibilities.
<!--more-->
1. you load the file completely and send it to the client
2. you load line by line and send each one to the client separately

## What are the advantages of choosing 2.

If the file is very large, this can quickly mean that the working memory is **not** sufficient to load the file for each request and keep it in memory. In the example you will use a CSV with more than 1GB. The backend could only handle as many requests in parallel as it can hold the data in memory. If you now stream the data, several clients can load the data at the same time. Of course it may take the first client a little longer to load all the data, but it can get much more data parralel.

Furthermore, the main memory will not run out of control, as only single lines will be kept in the memory.

## Getting the test data

First, you need test data for which streaming would be useful.

These wget from here:

`wget https://pysparksampledata.blob.core.windows.net/sampledata/sampledata.csv > sampledata.csv`

You load the file directly into a sampledata.csv. This data is about 1.34GB in size. With 16GB RAM we would not be able to handle 12 requests at the same time.

## Streaming with Spring Boot

Now you still have to read in these data as a stream. For this we create a *DataProvider*:

```kotlin
@Component
class DataProvider {
    /**
     * create flowable from sampledata.csv
     * @return every 100ms one line
     */
    fun getDataStream(): Flowable<String> {

        val csvFile = this::class.java.getResource("/static/sampledata.csv").openStream()

        return Flowable.using(
                { BufferedReader(InputStreamReader(csvFile)) },
                { reader -> Flowable.fromIterable<String>(getIterableFromIterator(reader.lines().iterator())) },
                { reader -> reader.close() })
    }
    /**
     * convert the iterator to iterable
     */
    private fun <T> getIterableFromIterator(iterator: Iterator<T>): Iterable<T> {
        return object : Iterable<T> {
            override fun iterator(): Iterator<T> {
                return iterator
            }
        }
    }
}
```

Here we read the *sampledata.csv* file from our static folder. The next step is to write this data as a stream. Therefore we create a component that inherits from *StreamingResponseBody*.

```kotlin
@Component
class CSVStreamResponseOutput : StreamingResponseBody {

    var dataProvider: DataProvider

    constructor(dataProvider: DataProvider) {
        this.dataProvider = dataProvider
    }

    /**
     * writes every line from the dataprovider to the output
     */
    override fun writeTo(os: OutputStream) {
        val writer = BufferedWriter(OutputStreamWriter(os))
        val countDownLatch = CountDownLatch(1)
        dataProvider.getDataStream().subscribe({
            writer.write(it)
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

Now you’re almost there. We create the *RestEndpoint* and then we can test our application.

```kotlin
@Controller
class RestEndpoint {

    var streamResponse: CSVStreamResponseOutput

    constructor(csvStreamResponseOutput: CSVStreamResponseOutput) {
        this.streamResponse = csvStreamResponseOutput
    }

    @GetMapping(path = ["stream"], produces = [MediaType.APPLICATION_STREAM_JSON_VALUE])
    @ResponseBody
    fun getCSVAsStreamWithExtraEndpointForLocust(): StreamingResponseBody {
        return streamResponse
    }
}
```

Now you have everything to send a CSV file performant line by line to the client.

## Testing the application

If you build the application with mvn clean install you get an executable jar. Now you can start with `java -jar target/csvStream-1.0-SNAPSHOT.jar`.

You can test the application with a simple *curl*.

```bash
curl localhost:8080/stream
```

You should see the data rattling down on the console. Line by line and the point immediately starts providing data.

## Conclusion

As you have seen, it is very easy to stream a CSV file. The advantage is obvious. You don’t have to load the 1.3GB into RAM first, and you immediately get the first lines from the CSV. The Serive reacts immediately and the client is able to work directly with the data.

Many thanks for reading. I hope it helps you. You can find the code here. If you have any questions, please feel free to contact [me](/about).