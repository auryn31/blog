---
title: "HowTo: Getting started with AWS Lambda"
subtitle: ""
date: 2019-09-21T14:12:35+02:00
lastmod: 2021-05-25T14:12:35+02:00
draft: false
author: ""
authorLink: ""
description: ""

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
AWS Lambda has long been a topic I would like to deal with. Now I have watched and started several tutorials. But most of them were either too long or too short. They all showed how to upload the function, but not how to test it locally.

<!--more-->

Now I have my first functions online and want to show you how easy it is to get started.

In general, AWS Lambda is very easy to use and allows an easy start in Serverless Computing with the free Contignent. What AWS Lambda is and what you use it for can be found [here](https://aws.amazon.com/de/serverless/videos/video-lambda-intro/) and at Google.

## Getting started

First you need the tool [serverless](https://github.com/serverless/serverless). The tool allows you to deploy and use the lambda functions on the Amazon servers (and many more like Azure, Alibaba Cloud, etc). You can find the Amazon payment model [here](https://aws.amazon.com/de/lambda/pricing/).

You can install serverless with the command `npm install -g serverless`. If it was installed correctly, test it with `serverless --version`. The output should look like this:

```sh
Framework Core: 1.52.2
Plugin: 3.0.0
SDK: 2.1.1
```

Now you create a folder in which you want to create your function. For this example I will name the folder `first_serverless_function`. If you change now with the terminal into the folder you can start here with the first lambda function. For the tutorial I use `Node.js`, where the tool `serverless` also provides boilerplate code for Python.

To start with the first AWS Lambda function, you can use the `serverless` boilerplate code:

`serverless create -t aws-nodejs`

In the terminal the output should look like this:

```sh
Serverless: Generating boilerplate...
 _______                             __
|   _   .-----.----.--.--.-----.----|  .-----.-----.-----.
|   |___|  -__|   _|  |  |  -__|   _|  |  -__|__ --|__ --|
|____   |_____|__|  \___/|_____|__| |__|_____|_____|_____|
|   |   |             The Serverless Application Framework
|       |                           serverless.com, v1.52.2
 -------'
Serverless: Successfully generated boilerplate for template: "aws-nodejs"
Serverless: NOTE: Please update the "service" property in serverless.yml with your service name
```

Now two files have been created. One `handler.js` and one `serverless.yml`. The `handler.js` contains the function and the `serverless.yml` describes it for AWS. You can test this function locally or directly to AWS Deployen.

## Lambda lokal Testen

To execute the function you just created locally, it doesn't take much.

In the `serverless.yml` it is described that our function is called `hello` and uses the handler `hello` which is described in the `handler.js`. You can also simulate the function call with the `serverless` tool. For this you have to call the function as follows:

`serverless invoke local --function hello`

The answer should look like this:

```json
{
    "statusCode": 200,
    "body": "{\n  \"message\": \"Go Serverless v1.0! Your function executed successfully!\",\n  \"input\": \"\"\n}"
}
```

To create a simple `Hello World!`, you can rewrite the function like this:

```js
module.exports.hello = async event => {
  return {
    statusCode: 200,
    body: `Hello ${event.queryStringParameters.hello}!`
  };
};
```

Now the function expects a parameter `hello`, which must be given. If you call the function as described above, this will lead to an error, because you did not specify a parameter. So the local testing has to change so that the parameter is given, or the function intercepts the error. It is best to do both. The function that catches the error directly looks like this:

```js
module.exports.hello = async event => {
  var name = "World"
  if(event.queryStringParameters != undefined && event.queryStringParameters.hello != undefined) {
    name = event.queryStringParameters.hello
  }
  return {
    statusCode: 200,
    body: `Hello ${name}!`
  };
};
```

Now the call of just (`serverless invoke local --function hello`) should not be an error anymore and your response should look like this:

```json
{
    "statusCode": 200,
    "body": "Hello world!"
}
```

Now give the function a name (`serverless invoke local --function hello --data '{"queryStringParameters":{"hello": "Peter"}}'`). The response changes to :

```json
{
    "statusCode": 200,
    "body": "Hello Peter!"
}
```

Now you have created a function that returns `Hello NAME!`. You will deploy it in the next step.

## Deployment

To deploy the function you need an AWS account. You can easily create it [here](https://aws.amazon.com/de/account/).

Once you have created an account, you can retrieve the login data to allow `serverless` deployment. You can find them [here](https://console.aws.amazon.com/iam/home?region=us-east-1#/security_credentials). Then create the credentials file for `serverless` with `serverless config credentials --provider aws --key KEY --secret SECRET`. You can check if this worked with `cat ~/.aws/credentials`. Both the key and the secret should be output.

Now you have to define the endpoint. This happens in the `serverless.yml` in lines 66-69. Here you change the endpoint like this:

```yaml
functions:
  hello:
   handler: handler.hello
   events:
     - http:
         path: hello
         method: get
```

And already you are ready to deploy your function to AWS. You do this with `serverless deploy`.

The deployment takes about a minute. After that the output in the console looks like this:

```sh
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service first-serverless-function.zip file to S3 (313 B)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
....................
Serverless: Stack update finished...
Service Information
service: first-serverless-function
stage: dev
region: us-east-1
stack: first-serverless-function-dev
resources: 10
api keys:
  None
endpoints:
  GET - https://afhjdhfskdj.execute-api.us-east-1.amazonaws.com/dev/hello
functions:
  hello: first-serverless-function-dev-hello
layers:
  None
Serverless: Run the "serverless" command to setup monitoring, troubleshooting and testing.
```

Now you can access the function you just created under the URL listed above. Without parameters you should simply return a `Hello World!`. If you now call the URL with the parameter `hello`(`URL?hello=Peter`), the function should answer `Hello Peter!` correctly.

## Summary

In the tutorial you created your first AWS Lambda function, tested it locally and then deployed and executed it on the Amazon server. You are now ready to create more complex functions and run serverless. Besides, you only pay what you need and have a high scalability. And all in under 30 minutes.

I'm excited about AWS Lambda and hope to use it in production soon. Did you like the little introduction, let me applaud, do you have a question or a suggestion, please comment the article.
