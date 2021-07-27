---
title: "React and UI5 Web Components"
description: ""
date: 2021-07-26T13:10:08+02:00
lastmod: 2021-07-26T13:10:08+02:00
draft: false
author: "Auryn Engel"

tags: ["React", "Frontend", "TypeScript", "UI5"]
categories: ["Development"]

---

## Getting started with React.js and UI5 Web Components

Why am I even writing about a topic that has actually been covered often enough?

Because I google this setup over and over again 🤯. I google again and again how to set up a React App, how to use SASS or how to add SAP UI5.

Also I often search how to configure a proxy and how to deploy it all to cloud foundry.
So i'm actually writing this for myself as a reference. If you feel like it, feel free to use it as well.
If you have any suggestions for improvement or changes, feel free to share or contact me.

## Create from template

```bash
yarn create react-app <APPNAME> --template typescript
```

## Add UI5 Web Components

```bash
yarn add @ui5/webcomponents-react \
  @ui5/webcomponents \
  @ui5/webcomponents-fiori
```

## Add SASS support

```bash
yarn add node-sass -D
yarn add sass
```

## Add [React Router](https://reactrouter.com/web/guides/quick-start)

```bash
yarn add react-router-dom
yarn add @types/react-router-dom -D
```

## Add HTTP Client

```bash
yarn add axios
```

## Serve on Cloud Foundry as Static Content

`Manifest.yaml`:

```yaml
applications:
  - name: <APPNAME>
    memory: 64M
    instances: 1
    path: build/
    buildpack: https://github.com/cloudfoundry/staticfile-buildpack
```

To enable routes create a file named `Staticfile` in the root of the project next to the `manifest.yaml` with the following content:

```txt
pushstate: enabled
```

## Serve on Cloud Foundry as a SPA with Express.js and a Approuter

### Approuter

Create ann `approuter` directory with the following files:

`package.json`

```json
{
  "name": "approuter",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@sap/approuter": "^8.6.1"
  }
}
```

`xs-app.json`

```json
{
  "routes": [
    {
      "source": "^/api/(.*)$",
      "target": "$1",
      "destination": "API_DESTINATION"
    },
    {
      "source": "^/",
      "target": "/",
      "destination": "react-ui-web"
    }
  ]
}
```

### Server

Create a `server` directory with the following files:

`package.json`

```json
{
  "name": "react-ui",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@sap/xsenv": "^3.1.0",
    "@sap/xssec": "^3.1.1",
    "express": "^4.17.1",
    "express-history-api-fallback": "^2.2.1",
    "passport": "^0.4.1"
  }
}
```

`index.js`

```js
const express = require('express');
const fallback = require('express-history-api-fallback');
const path = require('path');

const app = express();
const port = process.env.PORT || 3000;
const root = path.join(__dirname, 'build');

app.use(express.static(root));
app.use(fallback('index.html', { root: root }));

app.listen(port, () => {
  console.log(`UI listening on port ${port}`);
});

```

### UI

The ui directory is your root, where you did the `yarn create react-app <APPNAME> --template typescript`.

Create a `manifest.yml` on the root:

```yaml
--- 
    applications: 
    -   name: react-ui-web
        path: server
        memory: 256M
        buildpacks:
        - nodejs_buildpack
    -   name: react-ui
        path: approuter
        memory: 256M
        buildpacks:
        - https://github.com/cloudfoundry/nodejs-buildpack
        env:
            destinations: >
             [
                {"name":"react-ui-web",
                "url":"https://react-ui-web.cfapps.eu10.hana.ondemand.com",
                "forwardAuthToken": true}
             ]
        services:
        - API-SERVICE
```

### Makefile

```Makefile
deploy:
	rm -rf ./server/build && \
	yarn build && \
	cp -R build/ server/build && \
	cf push
```

### Start the App

Run `make deploy` and let the application run on CF.

### Add Security to Server

Connect the UAA service and update the `index.js`:

```js
const express = require('express');
const fallback = require('express-history-api-fallback');
const path = require('path');
const xssec = require('@sap/xssec');
const xsenv = require('@sap/xsenv');
const passport = require('passport');

const app = express();
const port = process.env.PORT || 3000;
const root = path.join(__dirname, 'build');

const services = xsenv.getServices({ uaa: 'UAA-SERVICE' });
passport.use(new xssec.JWTStrategy(services.uaa));
app.use(passport.initialize());
app.use(passport.authenticate('JWT', { session: false }));

app.use((req, res, next) => {
  if (req.authInfo) {
    next();
  } else {
    res.status(401);
    res.send('Unauthorized - make sure you are logged in');
  }
});

app.use(express.static(root));
app.use(fallback('index.html', { root: root }));

app.listen(port, () => {
  // eslint-disable-next-line no-console
  console.log(`UI listening on port ${port}`);
});

```

## Conclusion

Have fun 🥳
