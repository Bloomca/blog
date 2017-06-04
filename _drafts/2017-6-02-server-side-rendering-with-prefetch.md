---
layout: post
title: Server Sider Rendering with Prefetch
keywords: react.js, react, redux, server-side rendering, prefetch, javascript
---

## What is a server-side rendering

Server-side rendering (I'll use SSR later for the sake of brevity) is a pretty recent term, it started it's life just couple of years ago. Initially the main problem was lack of SEO for complex single-page applications, and projects like [prerenderer]() appeared. The main idea of it was prerender of the application somewhere else (e.g. PhantomJS), with waiting of the execution of the javascript, and then just grabbing all the html, and serving it later for robots. With this approach you updated your pages just sometimes (maybe in the background), and they were served only for search engines, so direct users were not impacted at all.

After react came out, with one of the principles that whatever is rendered inside it, should be able to be processed on the server side (or whatever side actually, later it allowed to write react-native and react-iot renderers), it became clear that it is possible to render all the content on the node.js, and then just pass directly to the user. Later other frameworks picked this idea (for example, Vue.js has pretty similar concepts of rendering), so nowadays SSR is considered as a good practice, rather than some rocket science (as it kind of was 2 years ago).

## Why do we need it?

This is actually a pretty good question -- crawlers are [much better now](), so your content should be parsed anyway, computers became much faster (as well as mobiles, so nobody really makes dedicated mobile websites nowadays, except for some rare and specific cases). Also, there are still no really good library solutions -- you would likely have to wire some things by yourself (or use some boilerplate, but it might bring you another kind of problems). Also, official starter kit from Facebook does not contain server-side rendering. So, why all the fuss?

Well, 
