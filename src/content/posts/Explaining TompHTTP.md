---
title: 'Explaining TompHTTP'
published: 2024-09-06
description: 'Explaining TompHTTP by implementing the Bare Server specification'
image: ''
tags: ['TompHTTP']
category: 'Programming'
draft: false 
language: 'English'
---

## What is TompHTTP?
TompHTTP (specfically the Bare Server) is a unified backend for webproxys to use.

### What is the backend used for?
The backend of a webproxy is used for fetching and returning HTTP requests.\
This is because web-filters (Things like Securly, Firewalls etc.) will block your request to a site if the site is not allowed, so webproxys use the Bare Server for making these requests.\
However it doesn't end there, the Bare Server also acts as a CORS proxy.

### What is CORS?
CORS (Cross Origin Resource Sharing) is a set of HTTP headers that allow a server to detact of the origin does not match its own.\
For example lets say I own https://example.com and Alice owns https://foo.bar, if Alice trys to use a asset from https://example.com and I have CORS setup the asset won't load on https://foo.bar

### How do we bypass CORS?
Lets take this NodeJS code:
```js
const express = require('express');
const axios = require('axios');
const app = express();
const port = 3000;

app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', '*');
  res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
  res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
  if (req.method === 'OPTIONS') {
    return res.status(200).end();
  }
  next();
});

app.use(express.json());

app.all('/proxy', async (req, res) => {
  const targetUrl = req.query.url;

  try {
    const response = await axios({
      method: req.method,
      url: targetUrl,
      data: req.body,
      headers: { ...req.headers, host: new URL(targetUrl).host },
      params: req.query,
    });

    res.status(response.status).send(response.data);
  } catch (error) {
    res.status(error.response ? error.response.status : 500).send(error.message);
  }
});

app.listen(port, () => {
  console.log(`CORS proxy server running on port: ${port}`);
});
```

How this works is that on the endpoint `/proxy` it has a URL paramater called `?url=`\
It will then take that URL and fetch it, however it modifies the headers to include:
```js
res.header('Access-Control-Allow-Origin', '*');
res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
```
These headers added will bypass CORS as it sets the CORS headers to show that all origins are allowed.

## Lets make our own Bare Server!
the TompHTTP Bare Server includes a specification so anyone can write there own Bare Server in any language they want.\
Today I'm going to be implementing this specification in NodeJS (In JS, not TS :3)

### Getting Started
Open up your terminal and run:
```sh
mkdir bare-server
cd bare-server
npm init
# If you don't have pnpm installed run npm i -g pnpm
pnpm i express
pnpm i -D @titaniumnetwork/ultraviolet-dev@2.0.0
```

Then create a directory called `static`\
Inside of this directory create the following files:
- `sw.js`
- `register-sw.js`
- `index.html`

While you're at it create a directory inside `static` called `uv` and created a file named `uv.config.js`\
Inside of `uv.config.js` paste in this:
```js
self.__uv$config = {
    prefix: "/uv/service/",
    bare: "https://localhost:8000/bare/",
    encodeUrl: Ultraviolet.codec.xor.encode,
    decodeUrl: Ultraviolet.codec.xor.decode,
    handler: "/uv/uv.handler.js",
    client: "/uv/uv.client.js",
    bundle: "/uv/uv.bundle.js",
    config: "/uv/uv.config.js",
    sw: "/uv/uv.sw.js",
  };
```

Paste this into `sw.js`:
```js
importScripts("/uv/uv.bundle.js");
importScripts("/uv/uv.config.js");
importScripts("/uv/uv.sw.js");

const sw = new UVServiceWorker();
self.addEventListener("fetch", (event) => {
 if (
    event.request.url.startsWith(location.origin + __uv$config.prefix)
  ) {
    event.respondWith(sw.fetch(event));
  }
});
```

`register-sw.js`
```js
if ("serviceWorker" in navigator) {
    window.addEventListener("load", () => {
        navigator.serviceWorker.register("/sw.js", {
            scope: "/"
        });
    });
}
```

`index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <script src="/uv/uv.bundle.js" defer></script>
    <script src="/uv/uv.config.js" defer></script>
    <script src="/register-sw.js" defer></script>
</head>
</html>
```

Then in the directory where you initalized your project create a file called `test-server.js` and paste in:
```js
import { uvPath } from "@titaniumnetwork-dev/ultraviolet";
import express from "express";

const app = express();

const publicPath = "static";
const port = 3000;

app.use(express.static(publicPath));

app.use("/uv/", express.static(uvPath));

server.on("listening", () => {
  console.log(`Listening on Port: ${port}`);
});

server.listen({
    port,
});
```

Once you have done everything you should have a file tree that looks similar to this:
```
- node_modules/
- package.json
- pnpm-lock.yaml
- test-server.js
- static/
  - uv/
    - uv.config.js
  - index.html
  - register-sw.js
  - sw.js
```

#### Extra
If you plan to upload this project to something like Github I suggest that you also do this.\
Inside your terminal enter your project directory and run `git init`

Create a file called `LICENSE` and paste in the AGPL-3 license text.

Create a file called `.gitignore` and paste in:
```
node_modules/
pnpm-lock.yaml
```

### Actually implementing the Bare Server
Create a directory called `src` and inside a file called `server.js`\
Inside the file paste in this:
```js
const info = {
    "versions": ["v1", "v2", "v3"],
    "langauge": "NodeJS",
    "memoryUsage": Math.round((process.memoryUsage().heapUsed / 1024 / 1024) * 100) / 100,
    "maintainer": {
        "email": "YOUR EMAIL",
        "website": "YOUR WEBSITE"
    },
    "project": {
        "name": "YOUR PROJET NAME",
        "description": "YOUR PROJECT DESCRIPTION",
        "email": "YOUR EMAIL",
        "website": "YOUR WEBSITE",
        "repository": "YOUR REPO",
        "version": "1.0.0"
    }
}
```
Using ExpressJS create the endpoint `/` that:
  - Returns the `info` `JSON` object
  - Sets the `Content-Type` response header to `application-json`

This should look something similar too
```js
app.get('/', (req, res) => {
    res.send(info);
    res.set('Content-Type', 'application-json');
})
```

okay I ran out of time and have to leave school I'll finish writing this once I do my like 7 missing assignments :3
