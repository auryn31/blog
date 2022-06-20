---
title: "Elm Dapp Gun Js"
description: ""
date: 2022-06-20T22:00:00+02:00
lastmod: 2022-06-20T22:00:00+02:00
draft: false
author: "Auryn Engel"

tags: []
categories: []

---
![compliments.dapp.auryn.dev](/img/compliments/compliments.png)

It all started with the video from [Fireship](https://www.youtube.com/watch?v=J5x3OMXjgMc) about [gun.js](https://gun.eco/). I was impressed that there was a way to make an app completely decentralized. Theme web3 and all.

Also to mention [this](https://www.youtube.com/watch?v=_eo_7BxTrmc) nice interview with Mark Nadal. This was eye-opening. He is a nice guy with a nice idea about the web. I completely agree with his ideas and hope, that someday, his ideas of a decentralized and user-owned web will be in place.

Now the question for me was, what exactly do I want to do completely decentralized, offline and with database and interaction?

A long time ago I wrote an app for compliments. The idea behind it was that you should just compliment other people every day. And since I'm relatively uncreative, I wanted an app for that. No sooner said than done.
Of course, the compliments should come from the heart, but on the one hand, they do that despite everything and on the other hand, you are happy, even if you know they are not from the other person's head, but from an app. And I can confirm that firsthand. I've been running the app for a while and have sent it to friends. They then sent me my own compliments and I was happy. Although they actually came from me.

However, since there are also limits to my creativity, I wanted everyone to be able to enter compliments. My last app is written with svelte.js and firestore. Here is a Google login before and everything ran great. But the data is now with Firestore and the login is also not in my hand.

So how would it be if the app was community-based? So with up- and downvote, own compliments you can enter and so on.

That's where Gun.js came in handy. The data is with the user, the app goes online and offline, and I don't have control over the data or votes. Sounds like a plan.

And since I'm currently writing everything with Elm, I wanted to do the same here. So functional as well. So everything a modern tech stack wants. Decentralized data storage, Web3, no user control from outside, functional programming. All buzzwords fulfilled. The only thing missing is crypto coins 🤣

## Web Code

For the website, the code is quite simple. You can send your own compliments, they go through a port to gun. How ports work I describe [here](https://blog.auryn.dev/posts/elm-aws-cognito-template/).

If new data comes from gun, it is automatically put into local storage and a port is used to send the new compliment or vote to elm and the UI updates itself accordingly.

The ports are in the [Gun.elm](https://github.com/auryn31/complimente-dapp-elm-gun/blob/main/src/Gun.elm) file.

Then there are two ports to use and copy the data via Telegram (currently my most used messenger). These can be found in the [Share.elm](https://github.com/auryn31/complimente-dapp-elm-gun/blob/main/src/Share.elm) file. The corresponding receivers and senders are defined [here](https://github.com/auryn31/complimente-dapp-elm-gun/blob/main/resources/entry.ts).

## Hosting

How to host the whole thing now? Since I host my website on a server anyway, I wanted to host the data there as well. Also, you need at least one node for Gun.js to sync. Since I didn't want to use a free open node, so that the data is cached for more than 15 minutes if the app is not used for a while, I use my own node. This is relatively easy to do with a Node.js server. The code can be found on the Gun.js page.

### Create Gun Server

Here is the whole express code to start the application:

```js
console.log("If module not found, install express globally `npm i express -g`!");
var port    = process.env.OPENSHIFT_NODEJS_PORT || process.env.VCAP_APP_PORT || process.env.PORT || process.argv[2] || 8765;
var express = require('express');
var Gun     = require('gun');

var app    = express();
app.use(Gun.serve);
app.use(express.static(__dirname));

var server = app.listen(port);
var gun = Gun({	file: 'data', web: server });

global.Gun = Gun; /// make global to `node --inspect` - debug only
global.gun = gun; /// make global to `node --inspect` - debug only

console.log('Server started on port ' + port + ' with /gun');
```

To prepare them, simply execute the following commands:

```sh
npm init
npm install express --save
npm install gun --save
```

#### Run express server

```sh
cd /root/compliments/compliments_dapp/server
nohup npm start &
disown
exit
```

or use [pm2](https://pm2.keymetrics.io/)

```sh
npm install pm2 -g
cd /root/compliments/compliments_dapp/server
pm2 start src/index.js
exit
```

I chose `pm2` because it is very simple and restarts the server even after a crash.

Now the whole thing must be made accessible from the outside. For this I use nginx. The configuration I have partly copied together, partly made myself.

## Create public route

To create a certificate for the server I use [certbot](https://certbot.eff.org/). It creates and manages the https certificates for me.

```sh
cd /etc/nginx/sites-available
touch <URL>

sudo ln -s /etc/nginx/sites-available/<URL>

sudo certbot --nginx -d <URL>

cd ../sites-enabled
```

```config

server {


    root <PATH_TO_STATIC_FILES>;

    index index.html index.htm index.nginx-debian.html;
    server_name <URL>;

    location / {
        try_files $uri $uri/ =404;
    }

    listen [::]:443 ssl; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    
    # IMPORTANT FOR WSS://
    location /gun {
        rewrite /gun/(.*) /$1 break;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-NginX-Proxy true;
        proxy_pass_header Set-Cookie;
        proxy_pass_header P3P;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_redirect off;
        proxy_pass http://127.0.0.1:8765; # final proxy pass to your localhost gun instance
    }
}

server {
    if ($host = <URL>) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80 ;
    listen [::]:80 ;
    return 404; # managed by Certbot
}

```

Create folder for static files:

`mkdir <PATH_TO_STATIC_FILES>`

## Build static files

```sh
yarn build
```

## Upload data to server

To upload the data to the server, i use [rsync](https://linux.die.net/man/1/rsync) to not upload all data every time.

`rsync -rz ./docs/ <SERVER_NAME>:<PATH_TO_STATIC_FILES>`

## Result

The result is a website that shows you the compliments that others have entered (attention, they don't come from me ;-) ). I also added a button to filter out the bad ones (5 or more downvotes more than upvotes). To add new ones, just enter this below and send. The data will be automatically sent to all other nodes and will be available. 🚨 Attention, they can not be edited or deleted 🚨. So be careful what you write.

If it gets too bad, I have to shut down the sync server ;-). So yes, I still have a little control, but if someone replaces my gun node, the control is completely gone.

I found it a funny project and am curious about what you add. Here is the URL now: [https://compliments.dapp.auryn.dev/](https://compliments.dapp.auryn.dev/)

Have fun :)

As it stands at the time of writing this blog:

![compliments.dapp.auryn.dev](/img/compliments/compliments.png)

And not to forget, a lot of thanks go to [deepl](https://www.deepl.com/translator), to help me with their nice translation tool to write my blogs :).
