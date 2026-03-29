---
layout: post
title: Learning Signals by creating one
keywords: JavaScript, Signals, Observables, tutorial
excerpt: "Signals are a simple composable state primitive, which enables powerful reactive principles."
---

If you ever heard of functional reactive programming (often referred as FRP) and were interested, you probably heard about observables. Time proved the concept to be extremely unwieldy and outside of some natural matches to be way too cumbersome, even though on paper it sounded really awesome. There is a much lighter and less dogmatic primitive, which is in [stage 1](https://tc39.es/process-document/) in TC 39 process -- signals. We won't explore the proposal here and simply look at what signals are and how they work underneath.

## What is a Signal

At its core, signal is simply a box for some value. This is not really interesting -- a simple object with the signature `{ value: any }` will do the same. To make it more useful, we can react to changes in that box by providing a callback. This is slightly more interesting, but nothing special still. A simple event emitter or an implementation of publish/subscribe pattern would allow us to achieve this part pretty easily.

What makes signals truly versatiles and helpful is composition. Think about the event emitter before: what would it take to get a value out of 2 event emitters and then somehow process it and to preserve the same flexibility? From my point of view, the great power of signals is that they are a composable reactive primitive.

## Creating a Signal

> While our version will work, some composition patterns would cause multiple triggers instead of one, but it should demonstrate the point pretty well.

Let's implement a simplified version of a signal. We'll start with the imperative APIs -- as I already mentioned, signals are boxes for the value inside, which can be read and set imperatively. The code is extremely straightforward:

```js
class Signal {
    constructor(initialValue) {
        this.#value = initialValue;
    }

    get() {
        return this.#value;
    }

    set(newValue) {
        this.#value = newValue;
    }
}
```

## Adding subscribers

The next step is to implement pub/sub functionality. Once again, the change is very simple:

```tsx
class Signal {
    ...

    this.#subscribers = []

    on(cb) {
        this.#subscribers.push(cb);
    }

    set(newValue) {
        this.#value = newValue;
        this.#subscribers.forEach(cb => cb(newValue));
    }
}
```

So far so good. In fact, so far we haven't achieved anything particularly great, but we've set up the groundwork for something really powerful -- ability to compose.

## Deriving signals

The key idea of composing signals is that the result of an operation which changes the signal is another signal. This allows us to pass signals around, combine them from multiple sources and transform existing signals. Let's start with a simple `map` operation (can also be named `transform` or `select`) -- simply change a signal value into something else using the provided function.

```
class Signal {
    ...

    map(fn) {
        const result = new Signal(fn(this.#value));

        this.on((newValue) => {
            result.set(fn(newVaue));
        })

        return result;
    }
}
```

Another powerful operation is combining multiple signals together.

```
class Signal {
    ...

    combine(...signals) {
        function calculate() {
            return [this.get()].concat(signals.map(signal => signal.get()));
        }
        const result = new Signal(calculate());

        [this].concat(signals).forEach(signal => {
            signal.on(() => {
                result.set(calculat())
            })
        })

        return result;
    }
}
```

## Diamond problem

Now that we can derive signals, let's create a simple example:

```
const sourceSignal = new Signal(1)
const firstDerived = sourceSignal.map(value => value + 1)
const secondDerived = sourceSignal.map(value => value * 2)
const combined = firstDerived.combine(secondDerived)
const final = combined.map(([first, second]) => first + second)
```

We added multiple manipulations