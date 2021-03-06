---
title: Managing Concurrency
description: Managing concurrency by abstracting concurrent task execution to make it feel synchronous.
layout: docs
---

* Table of Contents
{:toc}

The weak link when managing concurrency is humans; we simply don't think asynchronously or in parallel. Instead, we're really good at doing one thing at a time and the world around us generally fits this model. So to effectively design for concurrent processing in our code we have a couple of options:

1. Get smarter (not feasible);
2. Abstract concurrent task execution to make it feel synchronous.

## Promises

The basic unit of concurrency in an Amp application is the `Amp\Promise`. These objects should be thought of as "placeholders" for values or tasks that aren't yet complete. By using placeholders we're able to reason about the results of concurrent operations as if they were already complete variables.

> **NOTE**
>
> Amp promises do *not* conform to the "Thenables" abstraction common in javascript promise implementations. It is this author's opinion that chaining .then() calls is a suboptimal method for avoiding callback hell in a world with generator coroutines. Instead, Amp utilizes PHP generators to "synchronize" concurrent task execution.

### The Promise API

```php
interface Promise {
    public function when(callable $func, $cbData = null);
    public function watch(callable $func, $cbData = null);
}
```

In its simplest form the `Amp\Promise` aggregates callbacks for dealing with computational results once they eventually resolve. While most code will not interact with this API directly thanks to the magic of [Generators](#generators), let's take a quick look at the two simple API methods exposed on `Amp\Promise` implementations:


| Method                | Callback Signature                              |
| --------------------- | ------------------------------------------------|
| void when(callable)   | `function($error = null, $result = null, $cbData = null)` |
| void watch(callable)  | `function($updateData, $cbData = null)`                                   |


### `when()`

`Amp\Promise::when()` accepts an error-first callback. This callback is responsible for reacting to the eventual result of the computation represented by the promise placeholder. For example:

```php
<?php
$promise = someFunctionThatReturnsAPromise();
$promise->when(function($error = null, $result = null) {
    if ($error) {
        printf(
            "Something went wrong:\n%s\n",
            $e->getMessage()
        );
    } else {
        printf(
            "Hurray! Our result is:\n%s\n",
            print_r($result, true)
        );
    }
});
```

> **NOTE**
>
> We do not use type declarations here, as PHP 7 introduced the new `Throwable` interface and Amp is PHP 5 compatible.

Those familiar with javascript code generally reflect that the above interface quickly devolves into ["callback hell"](http://callbackhell.com/), and they're correct. We will shortly see how to avoid this problem in the [Generators](#generators) section.

#### Optional Callback Data

@TODO

### `watch()`

`Amp\Promise::watch()` affords promise-producers ([Promisors](#promisors)) the ability to broadcast progress updates while a placeholder value resolves. Whether or not to actually send progress updates is left to individual libraries, but the functionality is available should applications require it. A simple example:

```php
<?php
$promise = someAsyncFunctionWithProgressUpdates();
$promise->watch(function($update) {
    printf(
        "Woot, we got an update of some kind:\n%s\n",
        print_r($update, true)
    );
});
```

#### Optional Callback Data

@TODO

## Promisors

`Amp\Promisor` is the abstraction responsible for resolving future values once they become available. A library that resolves values asynchronously creates an `Amp\Promisor` and uses it to return an `Amp\Promise` to API consumers. Once the async library determines that the value is ready it resolves the promise held by the API consumer using methods on the linked promisor.

### The Promisor API

```php
interface Promisor {
    public function promise();
    public function update($progress);
    public function succeed($result = null);
    public function fail($error);
}
```

#### `promise()`

@TODO

#### `update()`

@TODO

#### `succeed()`

@TODO

#### `fail()`

@TODO

> **NOTE**
>
> We do not use type declarations here, as PHP 7 introduced the new `Throwable` interface and Amp is PHP 5 compatible.

### Deferred

`Amp\Deferred`  is the standard `Amp\Promisor`  implementation.

Here's a simple example of an async value producer `asyncMultiply()` creating a promisor and returning the associated promise to its API consumer. Note that the code below would work exactly the same had we used a `PrivateFuture` as our promisor instead of the `Future` employed below.

```php
<?php // Example async producer using promisor

function asyncMultiply($x, $y) {
	// Create a new promisor
	$deferred = new Amp\Deferred;

	// Resolve the async result one second from now
	Amp\once(function() use ($deferred, $x, $y) {
		$deferred->succeed($x * $y);
	}, $msDelay = 1000);

	return $deferred->promise();
}

$promise = asyncMultiply(6, 7);
$result = Amp\wait($promise);
var_dump($result); // int(42)
```

## Combinators

### `all()`

The `all()` functor combines an array of promise objects into a single promise that will resolve
when all promises in the group resolve. If any one of the `Amp\Promise` instances fails the
combinator's `Promise` will fail. Otherwise the resulting `Promise` succeeds with an array matching
keys from the input array to their resolved values.

The `all()` combinator is extremely powerful because it allows us to concurrently execute many
asynchronous operations at the same time. Let's look at a simple example using the amp HTTP client
([artax](https://github.com/amphp/artax)) to retrieve multiple HTTP resources concurrently ...

```php
<?php
use function Amp\run;
use function Amp\all;
use function Amp\stop;

run(function() {
    $httpClient = new Amp\Artax\Client;
    $promiseArray = $httpClient->requestMulti([
        "google"    => "http://www.google.com",
        "news"      => "http://news.google.com",
        "bing"      => "http://www.bing.com",
        "yahoo"     => "https://www.yahoo.com",
    ]);

    try {
        // magic combinator sauce to flatten the promise
        // array into a single promise
        $responses = (yield all($promiseArray));

        foreach ($responses as $key => $response) {
            printf(
                "%s | HTTP/%s %d %s\n",
                $key,
                $response->getProtocol(),
                $response->getStatus(),
                $response->getReason()
            );
        }
    } catch (Exception $e) {
        // If any one of the requests fails the combo
        // promise returned by Amp\all() will fail and
        // be thrown back into our generator here.
        echo $e->getMessage(), "\n";
    }

    stop();
});
```


### `some()`

The `some()` functor is the same as `all()` except that it tolerates individual failures. As long
as at least one promise in the passed array the combined promise will succeed. The successful
resolution value is an array of the form `[$arrayOfErrors, $arrayOfSuccesses]`. The individual keys
in the component arrays are preserved from the promise array passed to the functor for evaluation.

### `any()`

@TODO

### `first()`

Resolves with the first successful result. The resulting Promise will only fail if all
promises in the group fail or if the promise array is empty.

### `map()`

Maps eventual promise results using the specified callable.

### `filter()`

Filters eventual promise results using the specified callable.

If the functor returns a truthy value the resolved promise result is retained, otherwise it is
discarded. Array keys are retained for any results not filtered out by the functor.


## Generators

The addition of Generators in PHP 5.5 trivializes synchronization and error handling in async contexts. The Amp event reactor builds in co-routine support for all reactor callbacks so we can use the `yield` keyword to make async code feel synchronous. Let's look at a simple example executing inside the event reactor run loop:

```php
<?php

function asyncMultiply($x, $y) {
    yield new Amp\Pause($millisecondsToPause = 100);
    return ($x * $y);
}

Amp\run(function() {
    try {
        // Yield control until the generator resolves
        // and return its eventual result.
        $result = yield from asyncMultiply(2, 21); // int(42)
    } catch (Exception $e) {
        // If promise resolution fails the exception is
        // thrown back to us and we handle it as needed.
    }
});
```

As you can see in the above example there is no need for callbacks or `.then()` chaining. Instead,
we're able to use `yield` statements to control program flow even when future computational results
are still pending.

> **NOTE**
>
> Any time a generator yields an `Amp\Promise` there exists the possibility that the associated async operation(s) could fail. When this happens the appropriate exception is thrown back into the calling generator. Applications should generally wrap their promise yields in `try/catch` blocks as an error handling mechanism in this case.

### Subgenerators

@TODO Discuss `yield from`

### Implicit Yield Behavior

Any value yielded without an associated string yield key is referred to as an "implicit" yield. All implicit yields must be one of the following two types ...

| Yieldable        | Description                                                                                                                                                                                                                      |
| -----------------| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `Amp\Promise`    | Any promise instance may be yielded and control will be returned to the generator once the promise resolves. If resolution fails the relevant exception is thrown into the generator and must be handled by the application or it will bubble up. If resolution succeeds the promise's resolved value is sent back into the generator. |
| `null`      | @TODO |


> **IMPORTANT**
>
> Any yielded value that is not an `Amp\Promise` or `Generator` will be treated as an **error** and an appropriate exception will be thrown back into the original yielding generator. This strict behavior differs from older versions of the library in which implicit yield values were simply sent back to the yielding generator function.


### Extending Coroutine Resolution

@TODO Discuss custom promisifier callables
