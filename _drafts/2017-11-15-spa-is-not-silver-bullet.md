---
layout: post
title: Single-Page Application is not a silver bullet
keywords: javascript, SPA, gmail, application, native application, server-side application
---

I am a frontend developer, who all his career was involved into creating quite complicated single-page applications (like [Lebara Play](), for example). Some time ago, at my previous job, my manager came to me asking to investigate our application size -- about two years ago we migrated from WordPress to React/Redux/Immutable.js stack (which is a pretty common choice, I guess), and it turned out that average load time increased twice! I was looking into it for couple weeks, and after my research I re-thought a lot in front-end development. But let me start from the beginning.

## History Lesson

Historically, JavaScript was not a big deal -- all the content was generated on the server, and on the client at most some animations were added. Later browsers started to add more APIs to interact within JavaScript, and after adding [AJAX]() and using it for [Gmail](), people started to use it heavier. However, it was still far from wide adoption, and looked more like experiments for very specific use-cases, when people tried really hard to create "native-like" applications.

But potential was clear, and a lot of technologies were created, like frameworks -- Backbone.js, Knockout.js, Ember, Angular, React, Cycle.js and many more (there is even a joke about "0 days without new javascript framework"). Tooling was developing rapidly too, and after realizing that Node.js is the best choice (initially some tools, like Sass were written in ruby), a lot of building libraries appeared -- require.js, grunt, gulp, browserify, webpack and so forth (I know, these libraries are very different, but they all used to build your application).

Nowadays we ended up in [JS fatigue]() -- situation, when where are so many tools available and nobody really knows what is the best way to combine and configure them, that people are literally overwhelmed and don't know what to choose. However, there are projects like [create-react-app](), or [angular-cli](), which are trying to make all new projects in respective technologies look similar, and to hide complexity of configuring behind some internal libraries.

JavaScript is a different language nowadays, and a lot of new exciting stuff is coming, and technology allows us to build anything we want, purely on the front-end. We can even use some [serverless]() technologies to avoid having any traditional backend, just wiring severak SaaSes together!

## State of the Art

Nowadays a lot of startups, cool companies and personal websites are made using single-page applications -- it is pretty common, and nobody is particularly surprised when they don't see a reaload after clicking on a link inside the application.

Pros:

- no page reloads after clicking on the link
- with implemented caching subsequent pages open faster, because a lot of info can be already downloaded (like user info, movie details, etc)
- granular actions are super fast -- like add to the favourites, subscribe to the news; they can be fast
- frontend is decoupled from backend, makes development of complicated features much easier (especially when we need to interact between screens)
- possible to create almost "native" experience, with fallbacks if no internet is there

Cons

- broken "back" button (sometimes it works properly, but in general people don't trust it)
- broken "open in a new tab" behaviour -- people like to handle links in `onClick` handler, and browser can't recognize it as a link
- sometimes broken "refresh" button -- after refreshing you end up in a different UI (usually slightly, but still different)
- increased TTI (time to interaction), more on this later
- bad performance on low-end devices (and I suspect higher battery consumption, but can't really proof this one) 

## Why is it slow?

Back to the beginning, where I said we found that WordPress was actually faster than our new shiny stack. Our application was not the most sofisticated, but had some interactive parts -- it was a financial website, where you can apply for a loan, go to your personal accout, restructure your loan, and change some personal details. Because it is financial and fully online, we had to collect a lot of data, and it was several forms with live validation, on-the-fly 

Also, in order to help SEO and mobile clients, we had [server-side rendering]() implemented using Node.js + Express, where we fetched all needed data for the client.


## Conclusion

Do we really need that much SPAs? I don't know to be honest, but I highly doubt. Maybe this will 