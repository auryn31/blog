---
title: "How To: Ivy Lee Method with Notion and an E-Paper Display"
description: "In this article, I created an E-Paper Ivy Lee display to show how I want to increase my productivity."
date: 2021-05-31T22:45:00+02:00
lastmod: 2021-05-31T22:45:00+02:00
draft: false
author: "Auryn Engel"

tags: ["ESP32", "C++", "Arduino", "Productivity"]
categories: ["Development", "Productivity"]

---
The Ivy Lee method is a way to be more productive.
<!--more-->

At the beginning of the 20th century, Charles M. Schwab was one of the richest men in the world. he was president of Bethlehem Steel and was trying to achieve constant improvement in his company. when he was not satisfied with the productivity of his employees, he hired Ivy Lee.  A management consultant was supposed to look for a solution.

When Schwab asked him how he could create more, lee allegedly asked for just a few minutes with him and each of his managers. The price for his work: whatever it was worth to the company's president. Schwab was supposed to pay Lee only after three months - provided his method had worked.

And the method worked so well that Schwab paid him $25,000 as a thank you (today that would be about $400,000).

## The method

The method is quite simple. It consists of the following six steps:

1. write down the six most important things for the next day (or today if you are doing it in the morning)
2. prioritise these six items of their importance
3. concentrate only on the first task of the list, until it is finished
4. move to the next task and keep all not finished tasks for the next day
5. repeat this process every day

That's it. There is nothing more. Pretty easy, right?

If you want to find out more about the method and how it works, you can read more [here](https://jamesclear.com/ivy-lee).

## How I use it

To increase my productivity, I now also use the method. However, I don't apply it quite as stringently as described above.

The book [make time](https://maketime.blog/) by Jake Knapp and John Zeratsky takes up the method. They say, that you should only look for one highlight for the day and not be so strict about the other items you write down. However, you should place one task above everything else and work through it.

That's how I'm using it now. So it's a combination of make time and Ivy Lee. I write down five or six tasks for the next day and mark the highlight.

I use it in Notion. So here I have my list of elements and also a calendar with my tasks for the day. It looks something like this:

![How it looks like in Notion](/img/ivy-lee-e-ink-with-notion/example.png "How it looks like in Notion")

## Issue

But my problem is that I don't always have the tasks in front of me. Here, for example, [Ali Abdaal](https://www.youtube.com/channel/UCoOae5nYA7VqaXzerajD0lg) describes that he uses [Zettelkasten](https://www.youtube.com/watch?v=Bry8a_7b9aM).

So I don't always see my tasks or my highlight, but if I have it permanently in front of my eyes, I stick to it and always try to do my most important task.

So I thought about just using an e-paper display. And since Notion recently made its [API](https://developers.notion.com/) public, I can finally put my project into practice.

## E-Paper display

So I decided to use an e-paper display. In the first step, I use the Lilygo board, which is available for less than 20$. The ESP32 already has wifi integrated, which makes everything a little easier. The E-Paper display has 2.13 inch and can display about 7 lines. So perfect for my project.

## Getting Started

First, we need to create an integration and allow it to access the database with our tasks. This is described very well [here](https://developers.notion.com/docs), so I won't summarise it again here.

After we have the integration and the token, we should test if everything works. To do this, we run the following query with curl:

```bash
curl -X POST 'https://api.notion.com/v1/search' \
  -H 'Authorization: Bearer <YOU-SECRET-TOKEN>' -H 'Content-Type: application/json' \
  -H "Notion-Version 2021-05-13" \
        --data '{
    "query":"tasks",
    "sort":{
      "direction":"ascending",
      "timestamp":"last_edited_time"
    }
  }'
```

Now we should get all pages with the keyword 'tasks'. But we want to make our request on a database. This can look something like this:

```bash
curl -X POST 'https://api.notion.com/v1/databases/<YOU-DATABASE-ID>/query' \
  -H 'Authorization: Bearer <YOU-SECRET-TOKEN>' -H 'Content-Type: application/json' \
  -H "Notion-Version 2021-05-13" \
        --data '{
    "filter": {
        "property": "Date",
        "date": {
            "equals": "2021-05-31T02:43:42Z"
        }
    },
    "page_size": 5,
    "sorts": [
        {
            "direction": "descending",
            "property": "Highlight"
        }
    ]
  }'
```

For more detailed questions, you can read more [here](https://developers.notion.com/reference/post-database-query). The only important thing to know is that in Notion everything is a page. So even database entries are just pages with parameters.

## ESP32 Code

Here are the libraries I used with `platform.ini`:

```ini
[env:mhetesp32devkit]
platform = espressif32
board = mhetesp32devkit
framework = arduino
monitor_speed = 115200
upload_speed = 2000000
lib_deps = 
	zinggjm/GxEPD2@^1.3.3
	adafruit/Adafruit BusIO@^1.7.3
	adafruit/Adafruit GFX Library@^1.10.7
	bblanchon/ArduinoJson@^6.18.0
	arduino-libraries/NTPClient@^3.1.0
	paulstoffregen/Time@^1.6
```

And here is the code:

```cpp
#define ENABLE_GxEPD2_GFX 0

#include <GxEPD2_BW.h>
#include <GxEPD2_3C.h>
#include <Fonts/FreeMonoBold9pt7b.h>
#include <Adafruit_I2CDevice.h>
#include <WiFi.h>
#include <HTTPClient.h>
#include "ArduinoJson.h"
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <TimeLib.h>

#if defined(ESP32)
GxEPD2_BW<GxEPD2_213_B73, GxEPD2_213_B73::HEIGHT> display(GxEPD2_213_B73(/*CS=5*/ 5, /*DC=*/17, /*RST=*/16, /*BUSY=*/4)); // GDEH0213B73
#endif

#include "bitmaps/Bitmaps128x250.h"

const char *ssid = "WLAN SSID";
const char *password = "WLAN PASSWORD";
const char *db_url = "https://api.notion.com/v1/databases/<YOU-DATABASE-ID>/query";
const char *notion_secret = "Bearer <YOU-SECRET-TOKEN>";

const int text_height = 14;
const int rotation = 3;

int httpCode;
HTTPClient http;

// Define NTP Client to get time
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP);

void connectNetworkAndShowIP();
void loadTasksOfTheDay();
String getDateTimeDate();

void setup()
{
  Serial.begin(115200);
  Serial.println();
  Serial.println("Setup and connect WiFi");
  delay(100);
  display.init(115200);
  connectNetworkAndShowIP();
}

void loop()
{
  loadTasksOfTheDay();
  // wait for one hour to refresh
  delay(60 * 60 * 1000);
}

void connectNetworkAndShowIP()
{
  String tries = ".";
  WiFi.begin(ssid, password);
  display.setRotation(rotation);
  display.setFont(&FreeMonoBold9pt7b);
  display.setTextColor(GxEPD_BLACK);

  // display network to connect to and a busy indicator
  display.setFullWindow();
  display.firstPage();
  do
  {
    display.fillScreen(GxEPD_WHITE);
    display.setCursor(0, text_height);
    display.println(ssid);
    display.println(tries.c_str());
  } while (display.nextPage());

  while (WiFi.status() != WL_CONNECTED)
  {
    delay(1000);
    tries += ".";
    do
    {
      display.setPartialWindow(0, 2 * text_height, 250, 2 * text_height);
      display.fillScreen(GxEPD_WHITE);
      display.setCursor(0, 2 * text_height);
      display.println(tries.c_str());
    } while (display.nextPage());
  }

  // display local ip adress
  const char *ip_adress = WiFi.localIP().toString().c_str();
  display.setFullWindow();
  display.firstPage();
  do
  {
    display.fillScreen(GxEPD_WHITE);
    display.setCursor(0, text_height);
    display.println("IP address:");
    display.println(ip_adress);
    display.println("Loading Tasks...");
  } while (display.nextPage());

  timeClient.begin();
}

void loadTasksOfTheDay()
{
  // create filter for today and load data
  Serial.println("Loading Tasks:");
  String jsonData = "{\"filter\":{\"property\":\"Date\",\"date\":{\"equals\":\"" + getDateTimeDate() + "\"}},\"page_size\":5,\"sorts\":[{\"direction\":\"descending\",\"property\":\"Highlight\"}]}";
  http.begin(db_url);

  http.addHeader("Content-Type", "application/json");
  http.addHeader("Authorization", notion_secret);
  httpCode = http.POST(jsonData);

  display.setTextColor(GxEPD_BLACK);
  display.setCursor(0, text_height);
  display.setFullWindow();
  display.firstPage();
  if (httpCode > 0)
  {
    String payload = http.getString();
    DynamicJsonDocument doc(7172);
    deserializeJson(doc, payload);
    http.end();
    JsonArray results = doc["results"];
    String tries = ".";
    // Display the tasks for today
    do
    {
      display.fillScreen(GxEPD_WHITE);
      display.println("TASKS TODAY:");
      for (JsonObject v : results)
      {
        String todo = (v["properties"]["Name"]["title"][0]["text"]["content"]).as<String>();
        boolean highlight = (v["properties"]["Highlight"]["checkbox"]).as<boolean>();
        if (highlight)
        {
          todo = "* " + todo.substring(0, 19);
        }
        else
        {
          todo = "- " + todo.substring(0, 19);
        }
        display.println(todo);
        Serial.println(todo);
      }
    } while (display.nextPage());
  }
  else
  {
    do
    {
      display.fillScreen(GxEPD_WHITE);
      display.println("ERROR WHILE LOADING THE TASKS");
    } while (display.nextPage());
  }
}

String getDateTimeDate()
{
  while (!timeClient.update())
  {
    timeClient.forceUpdate();
  }
  unsigned long t = timeClient.getEpochTime();
  char buff[32];
  sprintf(buff, "%02d-%02d-%02dT%02d:%02d:%02dZ", year(t), month(t), day(t), hour(t), minute(t), second(t));
  String formattedString = buff;
  return buff;
}
```

Some useful links:

- [calculate the size for your DynamicJsonDocument](https://arduinojson.org/v5/assistant/)
- [ArduinoJson](https://arduinojson.org/)
- [Json to String Converter](https://tools.knowledgewalls.com/jsontostring)
- [Tutorial for Arduino POST requests](https://randomnerdtutorials.com/esp32-http-get-post-arduino/#http-post)

## Conclusion

The result looks like this:

![Result of the E-Paper](/img/ivy-lee-e-ink-with-notion/tasks.jpg)

I like the idea of having the tasks of the day always on my desk and without any disturbing light and nearly no energy consumption.

Next up is to update the code to make it more structured and more readable, but at the moment it does what I want and it updates my tasks of the day. I also want to draw a line through the already done tasks, but this is also the next step.

Thanks for reading. If you have any suggestions to improve the performance, the code or something else, let me know. Also feel free to share this post, if you liked the idea.
