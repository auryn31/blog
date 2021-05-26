---
title: "Add Cookies to Hugo Site"
description: ""
date: 2021-05-26T23:35:10+02:00
lastmod: 2021-05-26T23:35:10+02:00
draft: false
author: "Auryn Engel"

tags: []
categories: []

---
My blog is based on hugo. And since I use google analytics, I need a cookie notice to be gdpr compliant.
<!--more-->

But it was not so easy for me to find a page that describes how to do it. so that will be the content of this article.

## Getting started

The first thing we need is control over the theme. I use [PaperMod](https://themes.gohugo.io/hugo-papermod/), with which I am also very satisfied. so far.

So we forge the theme into our github and then use the fork as a submodule like this:

```bash
git submodule add https://github.com/auryn31/hugo-PaperMod.git themes/PaperMod --depth=1
```

Now we have control and can add code. I base myself on the tutorial by [Adam Spann](https://bas-man.github.io/post/add-cookie-warning/).

First we add the html part to the file `Papermod/layouts/_default/baseof.html`:

```html
<div class="cookie-container">
  <p>
    I use cookies on this website to give you the best experience on my blog. To find out more, read our
    <a href="/cookies/">privacy policy and cookie policy</a>.
  </p>
  <button class="cookie-btn">
    Okay
  </button>
</div>
<script>
    const cookieContainer = document.querySelector(".cookie-container");
      const cookieButton = document.querySelector(".cookie-btn");

      cookieButton.addEventListener("click", () => {
      cookieContainer.classList.remove("active");
      localStorage.setItem("cookieBannerDisplayed", "true");
      });

      setTimeout(() => {
      if (!localStorage.getItem("cookieBannerDisplayed")) {
          cookieContainer.classList.add("active");
      }
      }, 1000);
</script>
```

After that we have to style it. Here we simply work in file `PaperMod/assets/common/main.css`:

```css
.wrapper {
    padding: 2rem;
}

.cookie-container {
    position: fixed;
    bottom: -100%;
    left: 0;
    right: 0;
    background: #2f3640;
    color: #f5f6fa;
    padding: 0 32px;
    box-shadow: 0 -2px 16px rgba(47, 54, 64, 0.39);

    transition: 400ms;
}

.cookie-container.active {
    bottom: 0;
    display: flex;
    justify-content: space-evenly;
    align-items: center;
    flex-wrap: wrap;
}

.cookie-container>p {
    margin-top: .5rem;
}

.cookie-container a {
    color: #f5f6fa;
}

.cookie-btn {
    margin-top: 1rem;
    background: #e84118;
    border: 0;
    color: #f5f6fa;
    padding: .6rem 1.2rem;
    font-size: 1.2rem;
    margin-bottom: 1rem;
    border-radius: .5rem;
    cursor: pointer;
}
```

After that we only need one page for the cookies and then the result looks like this (can also be seen below 😋):

![cookie](/img/cookie/cookie.png)

I hope it helped you. As you can see, it is quite easy to create the gdpr comliance in Hugo.