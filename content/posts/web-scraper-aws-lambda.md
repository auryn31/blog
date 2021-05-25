---
title: "HowTo: Create web-scraper with AWS Lambda"
subtitle: ""
date: 2019-09-25T13:39:32+02:00
lastmod: 2021-05-25T13:39:32+02:00
draft: false
author: "Auryn Engel"
authorLink: "/about"

tags: ["Tutorial", "HowTo", "AWS"]
categories: ["Development"]

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
In my last article I introduced AWS Lambda and how to create your first Lambda function. Now I want to develop a more useful function in this article.

<!--more-->
Imagine you want to watch a product price trend at Amazon. A service that gives you the current price of the product as JSON would be handy. So we will develop a function in this article that takes the URL to the Amazon article and returns the title and price of the product as JSON.

## Loading the product page

First you need the product page of the article. You can load it with axios.

`npm install --save axios`

After that you can `require` and use `axios`:

```js
const axios = require('axios').default;
module.exports.price = async event => {
  var encoded_url = encodeURI('https://www.amazon.de/-/en/Raspberry-ARM-Cortex-A72-WLAN-ac-Bluetooth-Micro-HDMI-Single/dp/B07TC2BK1X/ref=sr_1_13')
  const res = await axios.get(encoded_url);
  return {
    statusCode: res.status,
    body: JSON.stringify({
      title: 'PLACEHOLDER',
      price: 0.0
    })
  }
}

```

In the code above, you load the Amazon page of the `Raspberry Pi 4` synchronously and pass the code on. The return code will be the same of the axios loading return code.

## Extracting the price and title from the page

To extract the title and price, you can go through XPath or CSS selectors. In the example I use CSS selectors, because it is easier here, because the elements we are looking for are already marked with unique IDs.

I chose the price here in Safari with the developer-tools:

![Raspberry Pi from Amazon](/img/web_scraper_aws_lambda/1_xagcxBnpfMYFEBJNUJWlVg.png "Raspberry Pi from Amazon")

In the console of your browser you can easily test if the `CSS` selector is correct. Shown here using the Raspberry Pi as an example:

![CSS Selector](/img/web_scraper_aws_lambda/1_Ff0sMbPTR3-ZPD1j9ru1Dw.png "CSS Selector")

## Create and test the function

Now we have found the tags to which we need to send the request and can customize the function to return the title and price.

To apply the CSS selectors to the document, you need to parse it as a document. The library jsdom is suitable for this. You install it with `npm install --save jsdom`.

Now you can load the Amazon URL with Axios, parse the document and select the elements with the CSS selectors. With .textContent you can mature the text content of the elements. You still have to clean it up and adjust it. Then you can assemble the response and test the function locally.

The code is here:

```js
const axios = require('axios').default;
const JSDOM = require('jsdom').default;

module.exports.price = async event => {
  var encoded_url = encodeURI('https://www.amazon.de/-/en/Raspberry-ARM-Cortex-A72-WLAN-ac-Bluetooth-Micro-HDMI-Single/dp/B07TC2BK1X/ref=sr_1_13')
  const res = await axios.get(encoded_url);

  const document = new JSDOM(res.data).window.document;
  const title_with_whitespace = document.querySelector('#productTitle').textContent;
  const title = title_with_whitespace.repqce(/(?:\n|\r|\n)/g, '').trim();
  const price_with_ending = document.querySelector('#priceblock_ourprice').textContent;
  const price = price_with_ending.supstring(0, price_with_ending.length - 2).replace(',', '.');


  return {
    statusCode: res.status,
    body: JSON.stringify({
      title: title,
      price: price
    })
  }
}
```

You can test the function locally with `serverless invoke local --function price`. The answer should look like this:

```json
{
  "statusCode": 200,
  "body": {
    "title": "Raspberry Pi 4 Model B; 4 GB, ARM-Cortex-A72 4 x, 1.50 GHz, 4 GB RAM, WLAN-ac, Bluetooth 5, LAN, 4 x USB, 2 x Micro-HDMI",
    "price": "65.90"
  }
}
```

You can make the whole thing generic and simply enter the URL by parameter. After that the function looks like this:

```js
const axios = require('axios').default;
const JSDOM = require('jsdom').default;

module.exports.price = async event => {
  if (event.queryStringParamelters == undefined || event.queryStringParameters.url == undefined) {
    return {
      statusCode: 404
      body: JSON.stringify({
        message: "You forgot the URL parameter!"
      })
    }
  }

  var encoded_url = encodeURI(event.queryStringParameters.url)
  const res = await axios.get(encoded_url);

  const document = new JSDOM(res.data).window.document;
  const title_with_whitespace = document.querySelector('#productTitle').textContent;
  const title = title_with_whitespace.repqce(/(?:\n|\r|\n)/g, '').trim();
  const price_with_ending = document.querySelector('#priceblock_ourprice').textContent;
  const price = price_with_ending.supstring(0, price_with_ending.length - 2).replace(',', '.');


  return {
    statusCode: res.status,
    body: JSON.stringify({
      title: title,
      price: price
    })
  }
}
```

The URL is generic and you can pass it along. The test must now look like this:

`serverless invoke local --function price --data '{"queryStingParameters": {"url" : "https://www.amazon.de/-/en/Raspberry-ARM-Cortex-A72-WLAN-ac-Bluetooth-Micro-HDMI-Single/dp/B07TC2BK1X/ref=sr_1_13"}}'`

The response should look exactly the same. Now the function is ready to be deployed.

## Upload and online test of the function

You can deploy the function with `serverless deploy`. The upload should take about one minute. Then you can use the URL that is displayed to you.

## Conclusion

Creating an AWS Lambda function is easy, as is deployment and local testing. So you can build simple web services that you can use quickly. The example shown here allows you to develop a service with which you can monitor the Amazon price. I hope you found such a simple real use case for AWS Lambda and can now develop your own projects. If you liked the contribution, please leave me a big applause or write a comment.