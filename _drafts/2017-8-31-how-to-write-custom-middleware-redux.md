---
layout: post
title: How to Write Custom Middleware for Redux
keywords: javascript, react, redux, redux middleware, redux-thunk, redux-saga, redux-promise, redux tutorial, redux middleware tutorial
---

Redux is a very popular library for global state management. It was inspired by bunch of projects, and was designed to be very low-level implementation of unidirectional data flow -- however, there is a very powerful extensibility point, [redux middlewares](). In the nutshell, they allow you to perform certain actions on the result of action calls, depending on the type of this result -- so, we can easily chain these middlewares.

## Redux logger

The simplest example for understanding is [redux-logger](). It is not that simple inside, but we will write our own version of it, really simplified.

```js
// let's start with a middleware which does nothing
// middleware is just a function which somehow processes
// action call result
function(action, next) {
  // just call next middleware
  return next(action);
}

// redux logger is side-effect only function, which
function(action, next) {
  log(action, next);
  return next(action);
}
```

## Redux-thunk

One of the most popular middlewares is one of the simplest one as well -- [redux-thunk](). It allows to return not just a plain object, rather a function, which will be invoked with `dispatch` and `getState` functions. Let's write one, and then compare to the actual implementation:

```js
function(action, next) {
  if (typeof action === 'function') {
    return action();
  }

  return next(action);
}
```

Let's compare it [with library]():

```js
...
```

As you can see, it is almost the same, except that you can actually instantiate it with additional parameters, which is very easy to add to our own implementation.

## Redux-promise

There are several implementations (for example, [1](), [2](), [3]()). I won't really get any as a reference here, but they still should be pretty similar -- this is still basic middlewares, and it is quite hard to come with something completely different.

```js
// duck typing! ()
function isPromise(action) {
  return Boolean(action && action.promise && action.promise.then);
}

function makeType(action, step) {
  return action.type + '_' + step;
}

export default function(action, next) {
  if (isPromise(action)) {
    dispatch({
      ...action,
      type: makeType(action, 'START')
    });

    return action.promise
      .then(data => {
        dispatch({
          type: makeType(action, 'SUCCESS'),
          payload: { ...action.payload, data }
        });
      })
      .catch(error => {
        dispatch({
          ...action,
          type: makeType(action, 'FAILURE')
          error
        })
      });
  }

  return next(action);
}
```

## Our Middleware with Cache

That all was very interesting, but what is the point? The goal of this article to make you not afraid of writing your own, custom middleware, which will solve your exact use-case. Let's imagine we want to achieve promise functionality, as in the previous example, but now with caching (so the same request, if it is in progress, won't be called again). Caching can be overriden, and we also assume we receive unique string identifier from action call.

```js
const cache = {};

function processAction(action, next) {
  const id = action.type;

  const promiseFromCache = cache[id];

  if (promiseFromCache &&
      // we can force one single request
      action.force !== true &&
      // or we can just omit caching, e.g.
      // for POST or DELETE requests
      action.noCache !== true) {
    return promiseFromCache;
  }

  dispatch({
    type: 
  });

  // return promise function is another approach
  // to implement promise middleware
  const newPromise = action.promiseFn()
    .then()
    .catch()
    // ES2017 feature 
    .finally(() => {
      cache[id] = null;
    });

  cache[id] = newPromise;

  return newPromise;
}

export function (action, next) {
  // for the sake of simplicity we assume that
  // this flag is also set up correctly by action
  if (action.ourCustomMiddleware) {
    return processAction(action, next);
  }

  return action(next);
}
```

## How to Use Our Middleware

You might say -- well, that all is very good, but we introduced a lot of additional fields which we need to remember and document, so additional work, will it pay off? This is a good point, and when you decide to roll out your own solution, always remember about trade-offs you have to consider.

The biggest advantage is that you can custom-tailor the solution to match your domain model. For example, if you operate only on collections, and you want to normalize your responses (using something like [normalizr]()), you can automate it using custom middleware -- in fact, you still can leverage other middlewares, just adjust object with data (or dispatch some other actions).

Another good example of custom middleware could be analytics, which will look into special properties and populate analytics data automatically, so you won't really have to remember about it each time, just implement one method `callWithAnalytics` and wrap all needed actions with it.

Let's try to write a small library for action calls to make use of our middleware a little bit easier:

```js
// using these functions, we have full control over the interface
// so if any other middlewares interfere with ours, we can change
// it without notice of end users
export function makeRequestWithCache(action, promiseFn, { force } = {}) {
  return {
    ...action,
    promiseFn,
    ourCustomMiddleware: true,
    force
  };
}

export function makeRequestWithoutCache(action, promiseFn) {
  return {
    ...action,
    promiseFn,
    noCache: true
  };
}

/**
Usage would be:
import { makeRequestWithCache } from '../our-middleware';

export function getUser() {
  return makeRequestWithCache({ type: 'GET_USER' }, ({ api }) => {
    return api.get('/me/details');
  });
}
*/
```

The single thing we did -- wrapped redux objects using our functions, so the end user does not have to remember all these flags, and concentrate only on actual meaning. It also gives us the ability to change the interface later and to support other features.

You can notice that our cache does not really work with parameters (like with `user_id`, for example), and is suitable only for simple actions, where we request same endpoint all the time. It is possible to extend, so in case you need to implement this part, don't be afraid to implement this logic and to simplify your actual business logic. Redux is [a very verbose library](), and I think writing custom middleware more often pays off than not.

## Possible caveats

Middlewares are pretty popular, but there is no real standard how to write them to avoid clashes, so it is actually possible. For example, redux-thunk just checks for a function, and in case your middleware also relies on this behaviour, they will not work together properly (one which is defined earlier, will hijack other's calls). So, be carefull about it -- I think, the safest bet is just passing objects with special properties in it, which are properly namespaced, so we avoid any possible issues this way. However, since you are the author, you will be able to resolve it pretty quickly, since you will be able to test and react much quicker, just keep in mind these possibilities.
