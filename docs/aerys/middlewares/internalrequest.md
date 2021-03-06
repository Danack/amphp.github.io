---
title: Using InternalRequest in Aerys Middlewares
description: Aerys is a non-blocking HTTP/1.1 and HTTP/2 application / websocket / static file server.
title_menu: InternalRequest
layout: tutorial
---

```php
(new Aerys\Host)
	->use(new class implements Aerys\Middleware {
		# Middlewares don't have to return a Generator, they can also just terminate immediately
		function do(Aerys\InternalRequest $ireq) {
			// set maximum allowed body size generally for this host to 256 KB
			$ireq->maxBodySize = 256 * 1024; // 256 KB
			$ireq->client->httpDriver->upgradeBodySize($ireq);

			// define a random number
			$ireq->locals["tutorial.random"] = random_int(0, 19);
		}
	})
	->use(function (Aerys\Request $req, Aerys\Response $res) {
		$res->stream("This is the good number: " . $res->getLocalVar("tutorial.random") . "\n\n");
		// send the body contents back to the sender
		$res->end(yield $req->getBody());
	})
;
```

Middlewares provide access to [`InternalRequest`](../classes/internalrequest.html). That class is a bunch of properties, [detailed here](../classes/internalrequest.html).

These properties expose the internal request data as well as connection specific data via [`InternalRequest->client`](../classes/client.html) property.

In particular, note the `InternalRequest->locals` property. There are the [`Request::getLocalVar($key)` respectively `Request::setLocalVar($key, $value)`](../classes/request.html) methods which access or mutate that array. It is meant to be a general point of data exchange between the Middlewares and the application callables.