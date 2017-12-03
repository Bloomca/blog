---
layout: post
title: How to Cancel Your Promise
keywords: javascript, promise, cancel promises, ES2015, ES6, async/await, modern javascript, generators, iterators
---

In ES2015, new version of EcmaScript, standart of JavaScript, we got new asynchronous primitive [Promise](). It is a very powerful concept, which allows us to avoid notoriously famous [callback hell]().

For instance, several async actions easily caused code like that:

```js

```

With promises we can refactor it to much more readable version:


> One of the drawbacks of using promise chains is that we don't have access to the lexical scope (or to variables in closure) of callbacks. You can read [a great article]() how to solve this problem from Dr. Alex ...

But, as soon it [was discovered](), you can not cancel a promise, and this is a real problem.

### Use Bluebird

[Bluebird] is a promise library, which is fully compliant with native promises, but which also adds couple of helpful functions to Promise.prototype. We won't cover them here, except for the `cancel` method, which does partially what we want from it -- it will never call any of `.then` or `.catch` handlers. Why is it only partially? The reason is that we can cancel only the whole promise, but not just part of it.

Consider the following example:

```js
function updateUser() {
  return fetchData()
    .then(updateUserData)
    .then(updateUserAddress)
    .then(updateMarketingData);
}

const promise = updateUser();
// wait some time...
promise.cancel(); // user will be updated any way
```

We have one function which returns us a promise. Imagine we want to cancel the promise, when we are updating address, at the second step. Just using `.cancel()` on the result will not affect the execution of the function, it will only cancel execution of all attached handlers. So, while this is pretty straightforward solution, it might help you.

### Pure Promises

In case you want to cancel execution of our update function, you can write pretty verbose version of our initial function, checking cancellation token at each step. In fact, we don't really need Bluebird for it, we can just reject with a special object, which will mark reason as cancellation. I will still use bluebird version, though, since it is more declarative:

```js
function updateUser() {
  let currentPromise = fetchData();
  const wrapWithCancel = (fn) => (data) => {
    // in case of native promises, we will keep
    // cancellation token in the closure, and
    // write something like:
    // if (!cancellationToken) {
    //   return fn(data); 
    // } else {
    //   return Promise.reject({ reason: "cancelled" });
    //}
    currentPromise = fn(data);
    return currentPromise;
  };
  return {
    promise: currentPromise
      .then(wrapWithCancel(updateUserData))
      .then(wrapWithCancel(updateUserAddress))
      .then(wrapWithCancel(updateMarketingData)),
    cancel: () => {
      currentPromise && currentPromise.cancel();
    }
  };
}
```

This is a little bit more verbose, and in case you need to repeat yourself pretty often, it becomes very annoying and easy to forget some use-cases. However, this thing will perfectly work, so in case you are using only promises (no generators and streams) and cancellation is needed in rare cases, feel free to take this approach.

### Switch to generators

Generators are another feature of ES2015, but they are not that popular for some reasons. Think about it, though, before adopting them -- will it be very confusing for your newcomers or you can deal with it? Also, they exist in some other languages, like [Python](), so it might be easy for you as a team to go with this solution.

So, generators provide you a way to "iterate" your function, controlling execution. "Steps" become keywords "yield", and we can write a simple function, which will execute generators, providing additional sugar, like `cancel` method in our case. 

```js
// this is a core function which will run our async code
// and provide cancellation method
function runWithCancel(fn, ...args) {
  // we should check the type, but I assume
  // everything is good here
  const generator = fn(...args);
  let cancel = false;
  const promise = new Promise((resolve, reject) => {
    let value;

    while() {
      // we can also just never resolve/reject, but
      // it can create memory leaks
      if (cancel) {
        reject("cancelled!");
        return;
      }
      
      value = generator.next(value);
    }

    resolve(value);
  });
  
  return {
    promise,
    cancel: () => cancel = true
  };
}

// * means that it is a generator function
// you can put * almost anywhere :)
function* updateUser() {
  const data = yield fetchData();
  const userData = yield updateUserData(data);
  const userAddress = yield updateUserAddress(userData);
  const marketingData = yield updateMarketingData(userAddress);
  return marketingData;
}

const { promise, cancel } = runWithCancel(updateUser);
```

As you can see, the interface remained the same, but now we have the option to cancel any generator-based functions during their execution for free, just wrapping into appropriate runner. The downside is consistency -- if it is just couple places in your codebase, then it will be really confusing for other people to discover that you are using all possible async approaches in a single codebase; it is yet another trade-off.

Generators are, I guess, the most extensible option, because you do literally everything you want -- in case of some condition you can pause, wait, retry, or just run another generator. However, I have not seen them very often in JavaScript code, so you should think about adoption and cognitive load -- do you really have a lot of use-cases for them? If yes, then it is a very good solution and you'll likely thank yourself in the future.

### Use streams (like RxJS)

Streams are completely different concept, but they are actually widespread [not only in JavaScript](), so you might consider it as a platform-independent pattern. They allow you to chain different operators (including network requests) in a very similar manner as promises, but they provide a method `takeUntil`, which basically behaves like a declarative cancel -- we just define at which conditions this stream should stop emitting values, and it will automatically, without our imperative code.

```js
// let's say we are waiting on actions
const updateUser$ = actions$.on('UPDATE_USER')
  .map(() => Rx.fromPromise(fetchData))
  .map((data) => Rx.fromPromise(updateUserData(data)))
  .map((data) => Rx.fromPromise(updateUserAddress(data)))
  .map((data) => Rx.fromPromise(updateMarketingData(data)))
  .takeUntil('ABORT_UPDATE_USER');
```

I am not a big specialist in using streams, but using them for a bit, I can say that you should either use them for all async stuff, or for none. So, in case you are already are using them, this question should not be a hard one for you, since it was a long time well-known feature of stream libraries.

### Note on async/await

In the version of [ES2017]() async/await were adopted, and you can use them without any flags in Node.js starting from the [version 8...](). 
Unfortunately, there is no support for cancellation at all, and since async functions return promise implicitly, we can't really affect it (attach a property, or return something else), only resolved/rejected values. On the other side, it is a standard feature of the language now, it has very convenient syntax, it allows you to use results of previous calls in the following (so problem of promise chaining is solved here), and has very clear and intuitive errors handling syntax via `try/catch`. So if cancellation does not bother you, then this feature is definitely a superior way to write async code in modern JavaScript.

### Accept it

Things are going to the right direction -- fetch is going to get [abort]() method, and cancellation is being under hot discussion for a long time. Will it result in some sort of cancellation? Maybe yes, maybe not. But also, cancellation is not that crucial for a lot of applications -- yes, you can make some additional requests, but it is a pretty rare case to have more than several consequent requests. Also, if it happens once or twice, you can just workaround those specific functions using the extended example from the beginning.
But in case there are a lot of such cases in your application, consider something from the listed above.
