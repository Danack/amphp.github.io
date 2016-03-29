---
title: Aerys
description: Aerys is a non-blocking HTTP/1.1 and HTTP/2 application / websocket / static file server.
title_menu: Introduction
layout: docs
---

`amphp/aerys` is a non-blocking HTTP/1.1 and HTTP/2 application, websocket and static file server written in PHP.

## Why ?

The computers we use are very fast, however most of their time is spent waiting for 'blocking IO' to complete. Traditional PHP web-servers/SAPIs are designed around a request coming in, waiting for it to complete, and then returning a response. 

Aerys uses 'non-blocking IO' to allow PHP code to be run asynchronously to give massively improved performance, compared to traditional PHP web-servers. With standard commodity hardware - Aerys can serve x thousand requests per second whereas traditional servers struggle to reach 1,000 requests per second.
[@TODO link to basic performance test code here]

## Optional Extensions

Although Aerys can run with just 'vanilla' PHP, using any of these extensions will allow it to have a much higher performance. 

- [ev](https://pecl.php.net/package/ev) provides interface to libev library - a high performance
full-featured event loop written in C.
- [libevent](https://pecl.php.net/package/libevent) - a wrapper for libevent - event notification library.
- [php-uv](https://github.com/bwoebi/php-uv) - an experimental interface to libuv for PHP

These extensions would typically see an improvement in performance from x requests per second for 'vanilla' PHP to y * x requests per second. These extensions aren't required to use Aerys; they are only needed if you really want to gain maximum performance for the hardware you're using.

## Current Stable Version

Aerys is currently still using 0.x tags as the API hasn't been fully locked down for a 1.0 release. However, the APIs are stable and only subject to very small changes that are required to solve bugs that are discovered, as we move towards a 1.0 release. 

## Installation

Aerys takes advantages of several new features in PHP 7, and that is the minimum version supported.

```bash
$ composer require amphp/aerys
```

## First run

```bash
$ vendor/bin/aerys -d -c demo.php
```

> **Note:** In production you'll want to drop the `-d` (debug mode) flag. For development it is pretty helpful though. `-c demo.php` tells the program where to find the config file.

## Blog Posts

 - [Getting Started with Aerys](http://blog.kelunik.com/2015/10/21/getting-started-with-aerys.html)
 - [Getting Started with Aerys WebSockets](http://blog.kelunik.com/2015/10/20/getting-started-with-aerys-websockets.html)

<div class="tutorial-next">Start with the <a href="setup/start.html">Tutorial</a> or check the <a href="classes">Classes docs</a> out</div>