---
layout: post
title: Introduction to generators
keywords: javascript, tutorial, generators, iterators, tutorial, ES2015, ES6, javascript generators, generators tutorial
---

JavaScript got a very neat feature in ES2015 standard -- [generators](http://www.ecma-international.org/ecma-262/6.0/#). In this tutorial I'd explain the basics and we will build a small library to have some advanced features, which were very hard to achieve before them.

## Basics

Generators are not a new concept -- they exist in other languages, like [Python](https://wiki.python.org/moin/Generators). But in JavaScript, with its [event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop), generators give you a lot of power, especially in async execution. 

You can think about generators as interactive debugger, where we put breakpoints using keyword `yield`, and then have a handle to hop to the next one.

So, let's see an example:

```js
// we can put * anywhere between `function` and actual name
// just stick with one version :)
function* fetchDataUsingGenerator(id) {
  const userData = yield fetchUserData(id);
  const userAddress = yield fetchUserAddress(id);
  const userSubscriptions = yield fetchUserSubscriptions(id);

  return {
    userData,
    userAddress,
    userSubscriptions
  };
}
```

##

To understand generators, let's create our own naive implementation. 

AAAA

library:

Returns you a promise, but allows to:
1. cancel your request!!!
2. pause/resume execution
3. 