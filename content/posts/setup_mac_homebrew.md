---
title: "Set up your Mac with Homebrew"
subtitle: ""
date: 2020-01-07T17:19:23+02:00
lastmod: 2021-05-24T17:19:23+02:00
draft: false
author: "Auryn Engel"
authorLink: "/about"

tags: ["Mac", "Setup", "Brew"]
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

Every time I get a new Mac, I get upset that I have to set it up again.
<!--more-->

Yeah, I could just use the time machine mackup. But this would bring me a lot of files and data on the new fresh Mac, which I don’t need there. A script would be much more comfortable here. One that 
installs and configures all programs and tools as I need them.

Now that I’m about to change my employer and know I need to set up a new Mac, I created one this time.

Or I’m about to create one.

## Idea

My ideas of a perfect script are the following:

- It should work without any pre-installed tools
- It should load all the programs I need
- It’s to configure the Mac the way I use it
- It is best to configure the installed programs already

Since I use brew a lot anyway, I thought this might be the way.

**Homebrew** is a free and open-source software package management system that simplifies the installation of software on Apple’s macOS operating system and Linux.

## Getting Started

I started my search by googling to see if that already exists. My first hit was from [codeinthehole](https://gist.github.com/codeinthehole). He published the following [Gist](https://gist.github.com/codeinthehole/26b37efa67041e1307db).

This gist contains a lot of apps, i already use. So I took this as a template and continued to search.

The second script comes from [Matthew Mueller](https://gist.github.com/matthewmueller) and contains many configurations with comments. It was linked in the script from codeinthehole. So I’ve done some research on this as well and copied some of his configurations into mine. You can find his Gist [here](https://gist.github.com/MatthewMueller/e22d9840f9ea2fee4716).

My changes can be seen in git. First of all, I kept looking for applications that I have on the current Mac and want to have back.

I have extended this one. But I noticed here with the script of codeinthehole that if an app that is listed was not installed with brew, the following apps are not installed anymore. But since I want to extend the script regularly and run it again, this should not be the case. So I wrote this in a for loop.

After adding all the apps, I went to the general configuration. Here the script from Matthew Mueller helped a lot. I copied almost all configurations I needed from his script. Some of them I still looked for myself 😅.

The last thing to do is to configure the apps. So mainly iterm, IntelliJ and vs Code.

This is done by mackup in combination with Dropbox. On the old Mac, i run `mackup backup`. All important configurations are loaded onto Dropbox. On the new Mac you only have to register Dropbox and with `mackup restore` the data are restored. This must be done after the script and the login into Dropbox.

After that, the new Mac should be ready to work.

But now enough, here is my [script](https://gist.github.com/auryn31/c5611eb41cce13a004044d57367188e1).

It will continue to develop and change over time. Therefore all recommendations I would make now may soon be outdated. If you like to use it, fork it gladly and let it go so far.

The script is under constant development, but normaly i only update it, if I have to setup a new mac. So please feel free to add suggestions or fork the script and create your own.

## Warning

**Warning** i didn't tested the script under the new macos. So everything is on your own risk.
