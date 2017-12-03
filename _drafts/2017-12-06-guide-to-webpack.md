---
layout: post
title: Guide to Webpack
keywords: webpack, javascript, bundling, babel, webpack tutorial, webpack guide, ES6, ES2015, typescript
---

Wepback is a big deal in modern web application development, and it is used on almost every big front-end project (and not only, actually, you can also build backend applications). But to configure it is [a very big problem](), and I am pretty sure, you've already heard a lot of complaints about its huge complexity.

In this guide I'll try to show how to create very simple configurations, which are nethertheless helpful.

> I assume that you use at least webpack@2

> this guide is about simple configurations. If you need to squeeze the maximum out of webpack, at first I recommend to stick with official tools, like [create-react-app]() or [angular-cli](), until you feel you definitely need more

## Start

Hello world for webpack is to get content of one file and write to another -- from entry to output. Let's do it; but first we need to set up our experiments directory:

```sh
mkdir webpack-tutorial
cd webpack-tutorial
npm init # you can just agree to everything :)
npm add -D webpack # save webpack to the dev dependencies
touch webpack.config.js # default filename for webpack config
```

Wonderful, now we are good to go! Open `webpack.config.js` in your favourite editor and type the following:

```js
const path = require('path');
module.exports = {
  entry: path.join(__dirname, 'index.js'),
  output: {
    name: 'build.js'
  }
};
```

This is enough for us. Now let's create the simplest `index.js` file in the root of our directory:

```sh
echo "console.log(1);" > index.js
```

Having this file, we can just run webpack. Because we did not install webpack globally, we don't have access to it from our command line. We have several options:

```sh
./node_modules/bin/webpack # same bin file if we install globally
npx webpack # requires node@5+
npm run build # requires modifying `package.json`, keep reading!
```

The first option is just a way to access bin file, the same one we will have access in case of global installation. Second approach is a new way of invoking binaries of npm packages, introduced in [node 5@+](). The last is to put a command to run webpack to `package.json`:

```json
{
  "name": "your-name",
  "scripts": {
    "build": "webpack"
  }
}
```

You'll need to create the `scripts` object in order to add these commands, and they will be available using `npm run %NAME%` command. These commands are run in your normal shell environment, but with added binaries from `./node_modules/bin`, that's why `webpack` is available without specifying path.

After you run your commands, you'll see newly created file `build.js`. If we look inside, we'll see webpack modules initialization code, and our `console.log(1)`. So, it worked! What is the point, though?

## Modules

Webpack@2+ supports modules [out of the box]() (and it was possible just to use `require` in webpack@1.x), and it means that you can modularize your application immediately.

> please note, though, that `import` and `export` are the only features, which webpack processes out of the box!

These several lines give us the ability to write normal JS code, but breaking application into small files and importing them where it is needed. Moreover, it will include only actually imported files in the build, and it will remove unused code (based on the usage of imports and exports it can detect what is defined, but never used).

So we got JS modules and tree-shaking for free, which is pretty cool!

## Multiple Entries

Another cool feature, which we get using webpack for free -- multiple entries. It means that the same transformation rules will be applied to several files, and in the output we will also get same number of files. Let's see an example:

```js
const path = require('path');
module.exports = {
  entry: {
    app: path.join(__dirname, 'index.js'),
    sdk: path.join(__dirname, 'sdk.js')
  }
  output: {
    name: 'build.[name].js'
  }
};
```

Now, if we create `sdk.js` file, after running our script, we will get two bundled files -- one for the application, and one for SDK. The main benefit of it is that if you need to bundle several applications which share a lot of functionality (or just to co-locate them), webpack allows to add this feature in just a few steps!

## Assets

While modules are cool, there are several other things each project can benefit -- assets. Webpack is basically a processing tool, which transforms all inputs by predefined rules -- there are some built-in, but for all others you'll need to download corresponding [loaders](). Let's start with default loaders -- they exist for JavaScript and for [JSON](). What does that mean? Each time we try to import a JSON file, webpack transforms it to a JS module, which has its object as a default export, and we treat it as just a usual JS file with a single object defined. 

Using [css-loader](), we can process CSS files as well (and there are all other loaders for preprocessors). What will be the export in this situation, though? It might be nothing (but the result will be included into final build), or resolved path to the asset, or it can be an object with class names, which later will be resolved by [css-modules]() to be hash-prefixed, but our code will magically use these class names using this imported hash!

So, webpack treats all imported files as something it has to transform -- if it has no idea how to process it, it will break and spit an error at you!

??? PICTURE WITH SPITTTED ERROR -- you might need to add a corresponding loader!!!

Let's add a simple loader for resolving our assets, namely images. This way we won't hardcode paths of images, we will just require them from our actual code, and we will get resolved path as a result of it:

```js
const path = require('path');
module.exports = {
  entry: path.join(__dirname, 'index.js'),
  output: {
    folder: path.join(__dirname, 'dist'),
    name: 'build.js'
  },
  resolve: {
    loaders: [
      {
        // take jpg and pngs by extension
        test: /\.jpg$|\.png$/,
        loader: 'file-loader'
      }
    ]
  }
};
```

> I've added folder to the output, because all our assets will also end up in this folder, and without this change we will add too much noise to the root folder

As you can see, the difference is not that big! Syntax is obscure, this is absolutely true (I use webpack for almost 2.5 years by now, and I had to look this syntax again), but don't be discouraged by it -- you just need to know several concepts, and then configure basic things looking them up. So, we add a universal [file loader](), which will put imported asset into final bundle, and resolve path correctly, which we can use in our code directly. It also gives us opportunity to keep assets close to the related files -- for example, if you have a component with some image, it makes sense to keep this image there as well!

So far we can import only pictures, but what else it can be helpful for? Actually, for a lot of stuff -- we can import files to download, like pdf files, excel sheets, word docs and so on. We even can require a css file, and put into `<link>` tag; however, it is not the best to require styles, which we'll cover in the next section.

Now we can write modularized JS code, directly requiring images and using them inside our code, avoiding hardcoding paths to our assets!

## Styles

Every frontend application has some styling, and webpack has a lot of loaders for this task, which make it easy to set up any processing you want. Styles are good example of chaining of loaders:

## Live Reload

We've gotten working application, but development experience is far from ideal -- we have to serve our bundle by ourselves, and after each change we have to reload our application manually.

> I won't cover [hot module replacement (HMR)]() here, because it is a pretty advanced topic, live reloading for the beginning is more than enough, from my point of view

All of these issues are addressed by separate packages -- there are a lot of possibilities, but we will cover the simplest one, [webpack dev server](). Webpack DevServer solves exactly our problems -- we can serve our application without external server, and dev server will reload the page for us as soon as something changes.

## Optimizations

We've gotten working application, but what is the point

## Tasks

Webpack has only task -- to bundle your application, and it does it very well: it takes entry point, and then resolves all possible dependencies by predefined rules, transforming input into (possibly) several final files. You might ask, what to do about regular tasks? Like what if I want to fetch updated translations and save it to some JSON file, which later will be consumed by our bundle?

The answer is that it does not lay in the responsibilities of webpack. You'll need to add something like [gulp task]() for that, or just an [npm script](), but it will be something different -- usually, in complicated projects webpack is just one of the steps, and there is [node.js API in webpack](), which you can use to do it from Node completely.