---
layout: post
title: How to Use Generators in JavaScript
keywords: javascript, generators, promise, async, cancel promise, pause, resume
---

Generators are very powerful concept, but it is not used that often (link to the twitter poll). Why is it so? They are more sophisticated than async/await, not so easy to debug (mostly back to old days), and people in general like async/await more, even despite that we can achieve similar experience in a really easy manner.

However, generators allow us to iterate over `yield` keywords using our own code! This is a super powerful concept, and we can actually manipulate execution! To start with not so obvious cancelling, let's start with synchronous operations.

## Batching (or Scheduling)

Generators return iterators, and it means we can actually iterate over it synchronously. Why would we want to? Well, the reason might be batching. Imagine that we download 10.000 items, and display them as rows in our table (don't ask me why, and let's assume we have no framework).
While there is nothing bad in showing them immediately, sometimes it might be not the best solution – maybe your MacBook Pro can handle it, but average people's laptops can not (not to mention mobiles). So, it means we need to somehow delay execution.

> Please note that this example is about performance optimization, and you should not do it until you actually hit this issue! Premature optimization [is evil](https://en.wikipedia.org/wiki/Program_optimization#When_to_optimize).

```js
function* renderItems(items) {
  // I use for..of iterator to avoid
  // new functions creation
  for (item of items) {
    yield renderItem(item);
  }
}
```

There is no difference, huh? Well, the difference here is that now we can run this function differently, without changing the source code – so it means that we can actually tweak it very easily. In fact, as I've mentioned before, there is no need to actually wait, we can execute it synchronously.
So, let's tweak our code. What about adding 4ms (one tick in JS VM) sleep after each yield? Well, given that it is 10.000 items, it will take 4 seconds – which is not bad, but let's say I want to render it in 2 seconds. The obvious answer is to chunk them by two, and all of the sudden the same solution using promises will start to become more complicated – we'll have to pass another parameter, number of elements in the chunk.

But we are using generators, so we can tweak only our runner.

## Cancellation

Let's go to more traditional stuff – cancelling. I've touched it extensively in my article about [promises cancellation in general](http://blog.bloomca.me/2017/12/04/how-to-cancel-your-promise.html), so I'll reuse code from it:

```js
function runWithCancel(fn, ...args) {
  const gen = fn(...args);
  let cancelled, cancel;
  const promise = new Promise((resolve, promiseReject) => {
    // define cancel function to return it from our fn
    cancel = () => {
      cancelled = true;
      reject({ reason: 'cancelled' });
    };
    
    let value;

    onFulfilled();

    function onFulfilled(res) {
      if (!cancelled) {
        let result;
        try {
          result = gen.next(res);
        } catch (e) {
          return reject(e);
        }
        next(result);
        return null;
      }
    }

    function onRejected(err) {
      var result;
      try {
        result = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(result);
    }

    function next({ done, value }) {
      if (done) {
        return resolve(value);
      }
      // we assume we always receive promises, so no type checks
      return value.then(onFulfilled, onRejected);
    }
  });
  
  return { promise, cancel };
}
```

## Pause/Resume

Another weird requirement might be pause/resume functionality. Why would you want it? Well, imagine we render our 10.000 rows, and it is pretty slow, and we want to give the power to pause/resume rendering to the user, so they can stop all background work and read already downloaded content. Let's do that:

```js
// our function is still the same!
function* renderItems() {
  for (item of items) {
    yield renderItem(item);
  }
}

function runWithPause(genFn, ...args) {
  let pausePromiseResolve = null;
  let pausePromise;

  const gen = genFn(...args);

  const promise = new Promise((resolve, reject) => {
    function onFullfilled(data) {
      next(data);
    };

    function onRejected(error) {
      reject(error);
    }

    const nextWithPromise = () => {
      if (pausePromise) {
        pausePromise.then(next);
      } else {
        next();
      }
    }

    const next = () => {
      const promise = gen.next();

      promise.then(onFullfilled, onRejected);
    };
  });

  return {
    pause: () => {
      pausePromise = new Promise((resolve) => {
        pausePromiseResolve = resolve;
      });
    },
    resume: () => {
      pausePromiseResolve();
      pausePromise = null;
    },
    promise
  };
}
```

Calling this runner will give us an object with pause/resume buttons, and all of that for free, using our previous function!

## Error Handling

We had this mysterious `onRejected` call, which is our main target for this section. If we use normal async/await or promise chaining, we fall through our try/catch, and it is really hard to recover to our point of failure without adding a lot of logic. Usual scenario – if we need to somehow handle the error (retry the call, for example), we just do it inside internal promise, which will handle itself and call the same endpoint again.
However, as usual, it is not a generic solution – but with generators we have the opportunity to affect it. Here we, though, hit the limitation of generators – while we can control the execution flow, we can not move around our generator function's body; so we can not return one step back and re-execute our command again. One possible solution is to use [command pattern](https://en.wikipedia.org/wiki/Command_pattern), which (unfortunately) dictates us the structure of everything we yield – it should be all info we need to execute this command, this way we will be able to execute it again. So, our function will be changed to:

```js
function* renderItems() {
  for (item of items) {
    // we had to pass everything:
    // fn, context, arguments
    yield [renderItem, null, item];
  }
}
```

However, we still can do something without affecting our function. For example, we can decide on strategy about ignoring errors – in async/await we'd have to wrap each call in try/catch, or add empty `.catch(() => {})` part.

```js
// code to ignore errors
```

You still can retry everything using command pattern as stated before, but it will add some obscurity, so you'll need to 


## Note About async/await

Async/await is a preferred syntax nowadays (and even [co]() tells about it), and it is a future. However, generators are inside the standard, and it means that in order to use them, you don't need anything, except writing several utility functions. I've tried to show you possible not so trivial examples, and it is up to you to decide whether it is worth or not. Remember, not so many people are familiar with generators, and in case you have only one place to use them in the whole codebase, it might be easier to workaround using promises.

Be wise in your choice – with great power comes great responsibility.
