---
layout: post
title: Overview of javascript build tools
keywords: javascript, node, npm, webpack, gulp, rollup, browserify, grunt, babel
---

Thanks to rising popularity of Node.js and development of [SPAs](https://en.wikipedia.org/wiki/Single-page_application), building of client-side projects are done exclusively using javascript ecosystem nowadays. In old days, in order to compile [sass](http://sass-lang.com/) into regular css, one had to have ruby installed, which introduced another dependency and another dotfile for version, and locking to a specific implementation of a compiler (so there is a chance of another dependency in the future, if some tool would be written in 3rd language). Switching to Node.js is not always a good thing -- for instance, if you have an old-school project, then you have to introduce a new dependency for you, Node.js. But at least from one point of view it is better -- because all best building tools are now written in JS, you can be sure that this new dependency will be only one.

The problem is, though, that, as usual for JS ecosystem, there are too many tools avaliable for kind of the same problem -- build your application: handle styles preprocessors, transpile modern JS to ES5, minify images and so on. I will try to list biggest of them, with their approach to solve problems and downsides and benefits. I'll also add code to process SASS and modern JS via [the babel](https://babeljs.io/) to ES5.

## NPM scripts

[NPM scripts](https://docs.npmjs.com/cli/run-script) were for some time a hidden feature of NPM, and used mostly for [publishing](https://docs.npmjs.com/misc/scripts) of packages, but not for actual managing of the project. Originally it was related to immature state of Node.js and lack of of modern EcmaScript features, so tools like [Grunt](https://gruntjs.com/) and [Gulp](https://gulpjs.com/) were much more popular. With time, though, Node.js started to offer a pretty convenient way to describe your tasks -- with mature ecosystem of packages and new features of modern javascript, and articles like the following started to appear:

- [Why I Left Gulp and Grunt for npm Scripts](https://medium.freecodecamp.org/why-i-left-gulp-and-grunt-for-npm-scripts-3d6853dd22b8)
- [Why npm Scripts?](https://css-tricks.com/why-npm-scripts/)
- [Comparison of npm scripts vs Gulp](https://gist.github.com/elijahmanor/179e47828bf760c218bb3820d929836d)

So, what _actually_ is `npm scripts`? It is just a regular JS code, which will be executed by Node.js. All packages, which do some transformation, offer a convenient interface nowadays, so you just write your code for it. It means that you have to combine tasks on your own, but thanks to [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises) and [asyc/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function), it is not that hard at all!
For `async/await` support, you need Node 8+, which will be a stable LTS version starting from October 2017, so you can use it right now.

-- code

When to use it? You will write pretty low-level code, so you need to understand on a decent level how these packages work and which arguments you need to provide. So, if you are comfortable with Node.js (and possibly with debugging), you need something specific and highly customizable (e.g. in Gulp, if you need to add a conditional, you need a [package](https://github.com/robrich/gulp-if) -- in Node.js it is just good old `if` in your code), then you can give it a try. You will need to dig through docs more, and sometimes maybe even go to source code, but in reward you'll get extremely custom-tailored setup.

## Grunt

[Grunt](https://gruntjs.com/) is not very popular anymore, but you can encounter it in old projects, in some legacy code, and so on. Grunt is a task-runner, and it means that you need first to describe and register a task, and then you will be able to run it. 

> Quick note about using bash. Sometimes you might encounter advice to actually use bash for tasks -- all these packages provide CLI, so you can actually do that! But the biggest problem here is that you leave holy land of JS (and not for the better replacement :)), so use this idea wisely

Another problem is that you start to depend heavily 
The biggest problem of grunt is that perform tasks sequentially, and there is no way to make it work in parallel. This naturally brings us to the next tool, gulp.

## Gulp

[Gulp]() is an essence pretty similar to previous tool, grunt. It is also a task runner -- we write a task, and it automatically registers itself, and then we can call it. In order to make it work, we also need plugins, 

The main difference is that gulp is based on [streams](), so it is not blocking at all. It is possible to run tasks sequentially, but the core idea behind is to write bunch of independent tasks (e.g. we independently minify png images and svgs)..

## Webpack

I put this one up because it seems to be one of the most popular bundler out there. So, what is a bundler, and webpack in particular? Bundler is much more than just a task runner. In fact, it is not really a task runner, so it is perfectly fine to use any of the previous tools in conjunction with webpack (or browserify).
In a nutshell, bundler handles _all_ your dependencies (including styles and images, yes), and pack them according to the rules you have defined -- it means that you require styles and images as a normal files, and in the output you will have javascript file, stylesheet file, and images, and links pointing to them will be correctly resolved in your JS (or CSS).

<img class="image" src="/assets/img/webpack_bundler.jpg" />
Graphical explanation of bundling

How does webpack actually work? It looks like a little bit of magic in the beginning, but it actually is not that special. It is a huge tool, so it is quite hard to grasp everything, but basics are more than possible.
Essentially, webpack does two things -- actual "bundling" and replacing "require" method.

#### Bundling

Bundling means "concatenating" your code. Javascript in the browsers has to be in a single file (and files between each other can communicate only via global variables), 


#### Hacking "require"

`require` is a global function in Node.js land, and 


Gulp, Grunt, browserify, webpack, rollup (other)