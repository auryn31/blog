---
title: "Downloading Files from a Website using Python"
subtitle: ""
date: 2021-05-24T16:54:55+02:00
lastmod: 2021-05-24T16:54:55+02:00
draft: true
author: "Auryn Engel"
authorLink: "/about"

tags: ["Python"]
categories: ["Development"]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: false
math:
  enable: true
lightgallery: true
license: ""


---
Yesterday I really wanted to do something with Python. Also a friend sent me a link, which contained many currently free computer science books from Springer. Now I didn’t want to downloading each book separately.
<!--more-->
So I thought to myself, how hard can it be to download the books automatically. And lo and behold, it wasn’t hard either. Enclosed, I would like to describe my learnings during the exercise and how it works that you can automate even simple web tasks.

To automate a website I use Selenium to control Chrome (you need to install ChromeDriver). So the first step is to create an instance of Chrome and call the corresponding URL.

![start](/img/downloading_files_using_python/start.png "Open Chrome")

The result then looks like this:
