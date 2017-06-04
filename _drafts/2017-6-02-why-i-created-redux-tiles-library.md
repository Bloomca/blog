---
layout: post
title: Why I created Redux-Tiles library to deal with Redux verbosity
keywords: react.js, react, redux, redux-tiles, server-side rendering, prefetch, javascript
---

Recently I published [Redux-tiles library](https://github.com/Bloomca/redux-tiles), which itself is a pretty small library intended to fight the verbosity of original style Redux. Redux is a great library, but in it's raw state it actually often frustrates people (especially newcomers).

This is an [example](https://github.com/Bloomca/redux-tiles/tree/master/examples/hacker-news-api) how it can be used inside a project, and this is the most important file, where we create all these redux entities:
<script src="https://gist.github.com/Bloomca/0942d38d9d366a9de0d84ca1141598b9.js"></script>

## What is Redux?

<img class="image" src="/assets/img/flux-diagram.png" />

Redux itself is an implementation of Elm-style app architecture, where we store state in an instantiated object, and then apply functions with specific type to this object, and another function, called reducer, reacts to this specific type. Such complicated architecture guarantees that data flow is unidirectional (just mutating this object won't change anything), and it makes it much more complicated to shoot in the foot. Initially it was presented on [React Europe 2015](https://www.youtube.com/watch?v=xsSnOQynTHs), and then it conquered the world (adaptions appeared in many languages). Nowadays it is a de-facto standard in React community, and even the creator (who was coincidentially hired by Facebook) tries to persuade people that [they might not need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367). Redux is heavily inspired by functional programming concepts, so some purists tried to put everything inside it, completely discarding local state.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">You Might Not Need Redux <a href="https://t.co/3zBPrbhFeL">https://t.co/3zBPrbhFeL</a></p>&mdash; Dan Abramov (@dan_abramov) <a href="https://twitter.com/dan_abramov/status/777983404914671616">September 19, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

So, while redux solves many problems (like updating number of notifications in different parts of the application), it also brings several new. One of the biggest problem is large amount of concepts, even for small things -- you have to create constants, actions and reducer; again, even in case if you want just to store some data. Also, imagine a situation, when you have 2 actions, let's say obtaining user's token and then authorization. Technically, both of these actions actually belong to another action -- logging in, and we might just do two requests in the same action (to save our time and to write less code). The problem here is that at some point we might need to do _only_ token request, and then things start to be more messy. Also, we might want to make UI more responsive, and actually provide status info, "checking your credentials" and "authorizing applications", and to do this we'd have to add at least one more whole package -- constant, action and reducer (we can handle second state checking first and loading of the login), but sooner or later it will bite us.

<script src="https://gist.github.com/Bloomca/81b2f489e96c604747be407792612c6c.js"></script>

## Problem's solultions

The problem is pretty well-known, and it is not about absence of solutions, rather absence of standard. The idea of modules is fairly common, for example [ducks-modular-redux](https://github.com/erikras/ducks-modular-redux) is pretty popular. Also, there are plenty of other solutions, but they are pretty customized. I personally think the problem lies inside [normalizr](https://github.com/paularmstrong/normalizr), which was popularized in the beginning of redux, and was often referred in initial tutorials hand by hand. It literally forces you to use redux as a database -- creating entities, parsing responses and putting ids instead of values. This is an interesting idea, and actually pretty good if you are working a lot with "collections" -- entities, which are often queried, have nested objects, and filtered by similar parameters; but it adds another magnitude of complexity for redux, and makes it literally impossible to somehow put common ground for all possible implementations.

I worked with different APIs, and with different codebase sizes, and my personal opinion is that if you have a big application with pretty specific API, you should create your own small library to grasp your domain. Sometimes it might be beneficial for you to inject inside normalizr behaviour, sometimes you'd need agressive caching, something you won't need it, but in general, it is much easier to custom-tailor in case your needs are pretty specific. You can use whatever data structures you want, parse data as you need, accumulate requests as you want, and so on.
So, with this knowledge in mind, let's try to list other possible options, for which we create redux modules:
- normal sync operations, which don't require api requests. For example, notifications -- we render them, and after closing we remove them from store
- usual async stuff, which we'd like to track -- loading, errors and successfull response
- nested variations of previous options -- when we'd like to put them by some separator:

{% highlight javascript %}
{
  documents: {
    terms: {
      isPending: false,
      data: {
        url: 'some',
      },
      error: null,
    },
  }
}
{% endhighlight %}

The whole purpose of redux-tiles is to solve all these 4 cases easily, to allow composition of modules, and to test code easily. Usually we keep almost all business logic inside redux modules, and it makes sense to inject all dependencies inside and test functions independently.

## Explanation

There is no real magic behind this library. Basically, it gets a configuration and based on it it creates actions, reducer, constants and selectors. It just gets desired nesting of the "tile" (module in the library's terminology), and function, to which it passes invoked params and all middlewares. In case of an async tile, it will expect a returned promise from a function, and will automatically populate next fields:

{% highlight javascript %}
{
  isPending: false, // will be true until the promise is resolved
  data: // data with which promise was resolved
  error: null, // if we throw an error from the promise
}
{% endhighlight %}

In case of sync tile, the library invokes a function with the same parameters, but expects just a value, not a promise to put into the state. Feel free to make asynchronous requests, but you should not wait for them.
Composition was one of the goals, so default middleware will inject `actions` and `selectors`, so you can use them inside functions, composing other tiles as you want.

So, here is a complete example how to use this library:
{% highlight javascript %}
import { createTile, createEntities, createMiddleware } from 'redux-tiles';
import { createStore, applyMiddleware } from 'redux';
import api from '../middlewares/api';

const clientInfo = createTile({
  type: ['client', 'info'],
  fn: ({ api }) => api.get('/client/info'),
  // info will be fetched only once
  caching: true,
});

const bookmarkInfo = createTile({
  type: ['bookmark', 'get'],
  fn: ({ api, params, actions, dispatch, selectors, getState }) => {
    await dispatch(actions.client.info());
    const clientInfo = selectors.client.info(getState());
    return api.get(`/bookmark/${params.id}`, { client: clientInfo.id });
  },
  // all bookmarks will be independent
  nesting: ({ id }) => [id],
});

const tiles = [
  clientInfo,
  bookmarkInfo,
];

const { actions, reducer, selectors } = createEntities(tiles);

createStore(
  applyMiddleware(
    createMiddleware({ api, actions, selectors })
  )
);
{% endhighlight %}