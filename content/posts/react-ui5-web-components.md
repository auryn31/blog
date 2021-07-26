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

## Add Loader

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

Have fun 🥳
