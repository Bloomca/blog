---
layout: post
title: Angular.js Guide for Seasoned Developers
keywords: javascript, angular, angular.js, angular.js tutorial, angular.js for react developers, directive, service, factory
---

This is a high-level overview of Angular.js (or 1st version of Angular), targeted to experienced JavaScript developers, after which you'll understand the concepts. It is not about API in details, you can read it afterwards -- my goal was to explain how Angular.js works so that you'll know what to expect from this framework. I assume you worked with Angular for a little bit, or at least skimmed over their docs.

Most guides for Angular are targeting beginners, and contain obsolete information. Also, nowadays nobody really talks about Angular.js (which is an official name for the first version), and I wanted to fill this gap, brief introduction with all concepts explaining, for experienced developers, with short examples and links to learn more.

While these links are helpful, though, I highly recommend to read the guide thoroughly first, and only after start to deep dive into the links (but feel free to use just links, if it suits you better!).

## TOC

- []
- []

## My experience

I've never worked with Angular in production before. I've heard terrible rumours, I've applied to certain jobs where it was a requirement, and I've played with it on my own (lack of links here tells for my success with it). At some point I've even learnt about digest cycle (we will learn about it soon!), but still had no idea how to actually write angular.js application. After some point Angular2 (nowadays 4) was released, and after that I just gave up on it.

But ironically on my new job we use Angular.js. Of course, it is officially admitted that it is a dead end, so we are migration into glorious React land, but for now a big chunk of the application is written in the Angular 1.5. After working couple months on it, I have to say that it is not that bad actually -- I'd say the biggest problem lies in allowing (and sometimes even encouraging) bad practices.

My experience was: `jQuery` -> `Backbone` -> `Knockout` -> `React` (with couple of custom frameworks around these tools in native JS). I am pretty experienced by frontend measure, about 5 years, and I decided to write this short guide how Angular.js works targeting people with similar path -- a lot of experience in JS and frontend, but not a lot of in Angular.

## Basics

Angular was built with extension of regular HTML in mind, so the simplest angular application is just a regular HTML page, but with a new weird `ng-*` attributes inside. The main directive `ng-app`, which is usually added on the `<html>` or `<body>` tags, tells Angular that it need to parse all markup inside. Angular parses everything, trying to find specific tags, attributes, classes or even comments (last two are usually frowned upon by community). After it encounters one, it does something depending on the type:

- if it is a custom tag, when it looks for registered directives (we will discuss them in details later), which can be used as tags, inserts its template and parses it, looking for other special markers
- if it is a custom attribute and it is a user-defined directives which applies to tags, this custom behaviour is added
- if it is a custom attribute with a built-in directive, then different behaviour is added. I won't cover all of them, and right now let's discuss only one -- `ng-controller`.

This is a simple example, but it goes beyond what we learned so far -- `{{ value }}` inside our markup. Now, let's look at controllers in more details.

```html
<body ng-app>
  <div ng-controller="myController">
    {{ value }}
  </div>

  <script>
    angular.module('app')
      .controller('myController', ['$scope', function($scope) {
        $scope.value = 'Value from the angular!';
      });
  </script>
</body>
```

## Controllers

Originally, controllers were the only way to add certain behaviour to HTML (I might be wrong here, as in any part of this article), so there are a lot of them, especially in legacy parts. New functionality usually is written using directives (which we'll cover soon!), but under the hood new controller is created each time any way (same thing for some built-in directives).

So, each time we use `ng-controller` attribute, Angular searches for the declared controller (via dependency injection, which we'll cover soon!), and creates a new instance with a `new` keyword. Following the best traditions, there are two ways to use it -- you can just set name of the controller, which you registered using Angular's dependency injection (more on it later), or you use [controllerAs syntax](). In case you use the latter, you can just treat controller as a construction function, which will be invoked with a `new` keyword, and all properties on `this` will be available inside template. `controllerAs` makes it clearer from which controller we want to get properties, so I personally like it more, but it is more about convention.

After we create instance, all variables declared on it (or on `$scope`, depending whether you declared using `controllerAs` syntax or not) are available in this scope. Scope is a magical angular thing, which gives you access to everything declared on it inside your HTML template. Controllers by default have parent scopes in their prototype chain, so it means that you can change properties in absolutely unpredictable places, and this is a really big problem (it is a widely recognized problem, just be super careful).

Also, remember that everything is inside some scope, and some built-in directives create a new one, and there is a common problem with declaring `ng-model` directive, which adds two-way data-binding from your scope to the input, when you change property in the child scope, which was introduced by another directive, but you actually wanted to change it inside your own, now parent directive!
I am pretty sure it sounds a little bit confusing, so please read much better and in-depth explanation of this issue: [aaa]()

As I mentioned before, each scope puts previous one in a prototypal chain. In practice it means that inside your HTML code each time you nest `ng-controller` attributes, they create a new scope, but have everything from parent scopes in prototype chain. It adds a lot of confusion, because it becomes unclear from where exactly value comes from, so in Angular 1.3 was introduced a special syntax [controllerAs](), which forces you to prefix all values with the name of controller. Better to illustrate:

```html
<div ng-controller="welcomeController" ng-controllerAs="welcome">
  {{ welcome.title }}
  <div ng-controller="subscribeController" ng-controllerAs="subscribe">
    {{ subscribe.title }}
    {{ welcome.user }}
  </div>
</div>
```

As you can see, there is no disambiguity, and we are 100% sure which scope is referred in each specific case. Nowadays directives (or components, which are just wrappers around directives) are preferred, but old code still uses controllers extensively.

## How Watching Works (or Digesting Cycle)

This is the most magical part of Angular, and also the biggest source of bugs, performance issues and all that hatred which Angular has.
Angular.js has two-way data binding by default -- you don't really need to do anything to make it work, you just update values on scope and it should reflect in HTML automatically. There are some quirks here, but the most important thing here is that by default magic works only if you do everything using angular directives/services, so they will cause HTML to re-render.

In the basics I've described how angular parses templates, but did not go into details. So, let's write super simple counter example and see how does it work:

```html
<body ng-app>
  <div ng-controller="counterController" ng-controllerAs="$ctrl">
    <button ng-click="$ctrl.increment()">
      Click to add!
    </button>
    <div>
      Number of clicks: {{ $ctrl.clicks }}
    </div>
  </div>

  <script>
    angular.module('app')
      .controller('counterController', function() {
        this.clicks = 0;
        this.increment = () => this.clicks++;
      });
  </script>
</body>
```

We added some interactivity here -- namely, number of clicks. Initial render will show us expected 0, but what will happen next? `ng-click` allows us to add click handlers, and as any other expression in angular, the value is evaluated using `eval`, so we have to write code like it will be executed inside click handler. If we click on this button, magically in our html we will see that number increased by 1. How does it work?

When angular sets up your controller in `ng-controller` or custom directive, after instantiating controller it will also set up all needed listeners. But how does it know that something has changed? In fact, it does not. Angular has its own [digesting cycle](), which shows the whole execution model of the framework. So, watchers just react to changes after special signal, called `$scope.$digest()` or `$scope.$apply()` (don't worry too much about difference between them -- you can read [this SO answer]() now or later about it) -- you can invoke it manually, actually, and sometimes you have to! You have to do so if angular has no idea what are you doing, and usually it means you don't use any angular directive/service at all.

??? PICTURE WITH DIGEST CYCLE FROM DOCS

For example, if we have a controller, which defines a variable, and then change it after some timeout, nothing will happen:

```html
<body ng-app>
  <div ng-controller="timeoutController" ng-controllerAs="$ctrl">
    {{ $ctrl.value }}
  </div>
  <script>
    angular.module('app')
      .controller('timeoutController', function() {
        this.value = 'Initial string';

        // this will not reflect in the DOM!
        setTimeout(() => this.value = 'String after timeout', 500);
      });
  </script>
</body>
```

So, previous example does not work, which makes us puzzled -- where is the magical two-way data binding? As I said, you have to force `$scope` to start digesting cycle, after which it will realize that `value` now has another value, and it re-render HTML. So, working controller from the previous example will look like this:

```js
.controller('timeoutController', ['$scope', function($scope) {
  this.value = 'Initial string';

  setTimeout(() => {
    this.value = 'String after timeout';
    // we trigger angular to re-render our template
    $scope.$digest();
  }, 500);
}]);
```

The problem is that now we have to write these updates manually all the time -- for Promises, for network requests, for timeouts, for global DOM events, for literally everything asynchronous! This is the reason why Angular provides a lot of services by default, which basically mimic existing functionality, but it wraps them in a function, which automatically invokes `$scope.$digest()` under the hood, so you don't have to do it all the time. Another good point is that it becomes extremely easy to mock everything, because we _everything_ is custom.

For timeouts we have [$timeout service](), which does exactly that -- it will execute your function, and then apply digesting automatically. The code will look like this:

```js
.controller('timeoutController', ['$timeout', function($timeout) {
  this.value = 'Initial string';

  // it will work as expected
  $timeout(() => this.value = 'String after timeout', 500);
}]);
```

So, keep in mind that if you don't use built-in angular services, or you have some bridge which consumes outside events, you will need to trigger these changes by yourself. Also, in case you have a lot of updates in the same [microtask]() -- it might be beneficial to use `$scope.$applyAsync()`, which will update in batches; but as in any performance-related topic, please test it first, and do premature optimizations.

- [Official docs about digest cycle]()
- [SO difference between `$apply` and `$applyAsync`]()

## Directives

It is the most important part of Angular.js, because right now it is a recommended way to build your applications (actually, [components]() are, but they are just a wrapper around directives). It allows you to create isolated directives, which receive only selected properties (both values and functions).

What is a directive? Directive a special angular object, which allows you to add custom tags and attributes to your templates. We learned in basics parsing cycle, and we did not stop on custom tags and attributes, except for recursive part -- let's cover it now. So, as soon as angular parser encounters custom tag (or attribute), it creates a controller for it, invokes [link]() function, and puts template into the DOM. Controller works exactly as we learned before, but there are some differences in scoping. By default it will inherit parent's scope, which is still the same; but we can pass `scope` argument with different arguments:

```js
{
  scope: true, // default option, inherits parent's scope
  scope: {}, // no inheritance at all
  scope: { // pick some properties
    name: '=name', // two-way data-binding with name property
    choose: '&select', // functions -- and prop name in HTML will be `select`
    isVisible: '<' // one-way data binding, property's name should also be `isVisible`
  }
}
```

As you can see, we can avoid scope in our prototype (you still can access it using [$parent](), but it is not recommended, except for debugging), and it is a huge step forward. The last example might be a little bit confusing, so let's see how we will render this directive in the template:

```html
<my-custom-directive
  name="user.name"
  select="selectLanguage" # this will be `choose` in our directive!
  isVisible="user.hasLanguages"
></my-custom-directive>
```

This concept of allowing certain properties is very similar to properties in any components-based frameworks, like React or Vue!

`controller` property allows us to define properties on scope, and works exactly as you expect -- it will be applied to our template automatically, and we also can use `controllerAs` syntax. What about `link` function? Why do we need it? Officially, we have to use `link` in case we want to add some side effects to the DOM -- we have access to both element and its attributes. For instance, if you want to add integration with some jQuery plugin, `link` function is the right place to do so -- you can attach any listeners to element, read any other attributes, add new attributes and so forth. Remember, though, that if you want to re-render your directive, in case updates come from external, non-angular source, you will have to `$apply` these changes.

`link` function receives scope as one of its arguments, and you can define its properties in it as well as inside the controller. Which one to choose? As I understand, if you don't need to interact with DOM, use `controller`, and otherwise use `link` function; however, it might vary depending on your team.

Another exciting (yet confusing for some people) feature is transclusion (directive `ng-transclude`) -- passing another template to our directive. Template string per se are just HTML, but this feature allows you to pass dynamic templates! Template which we are passing, is executed in the context where it was defined; it means you can create different wrappers around your content, without any coupling! You might think about it as `this.props.children` in React world -- it is a very similar concept.

The last confusing part is `compile` function, which by far is the most confusing thing. You don't really need to use it unless you need some performance optimizations -- it is invoked once, during registering your directive, so if you need to set up something, you can do it here.
??? COMPILE, PRE-LINK, POST-LINK

So, let's put everything everything together (except `compile` :)), and write a simple directive, which will be a form wrapper:

```js
angular.module('app')
  .directive(
    'appForm', // please note the camelCase -- in HTML we will use `app-form`!
    [function() {
      return {
        restrict: 'E', // allow only tags to use this directive
        transclude: true, // allow children
        replace: true, // <app-form> tag itself will be removed from DOM
        scope: {
          onSubmit: '&',
          name: '<'
        },
        controllerAs: '$ctrl',
        template: `
          <form name="">

          </div>
        `
      };
    }])
```


- [Official docs on directives]()
- [Difference between controller and link in directives]()
- [SO question about compile]()
- [SO question about the difference between directive and controller]()
- [SO question about directive's lifecycle]()

## Services and Factories

I will tell you immediately, that effectively there is no difference between them at all. Moreover, `angular.service` is defined in terms of `factory` inside, and in pseudocode we can write something like this:

```js
angular.service = function(name, Constructor) {
  return angular.factory(name, function(..args) {
    return new Constructor(...args);
  });
}
```

As you can see, the only difference is that function for `service` is a constructor one, and for the `factory` just a plain function. That is it about them, and since `factory` is more flexible, I can recommend to use only it; but if your team preferres services, it is perfectly fine, just stick with one, since having more entities doing exactly the same is a little bit confusing.

You should treat services as a tool how different components can execute same actions. E.g. if we have user entity in our application, it makes sense to put all actions related to user -- update personal data, upgrade subscription plan, add device, etc, to a unique service. Also, we can get require all other services inside, so don't be afraid to separate network layer to another service, as well as some specific logic, like authentication, for example.

Let's write several services/factories:

```js
// all these services should be each in unique file
// but for the sake of simplicity, we put them together
angular.module('app')
  .factory('Api', ['$http', function($http) {
    return {
      get: 
      post:
    }
  }])
  .factory('User', ['', function() {
    
  }])
  .factory('Authentication', ['', function() {

  }]);
```

Also you can put other libraries to services as well, and it will make testing easier

> Directives are also nothing else but factories, which return object of a special type

## Dependency Injection (DI)

Angular.js has its own pretty famous dependency injection system, which makes some things easier (e.g. it is possible to mock literally everything, so testing won't be a problem). 

## Testing

All of the sudden, good part of Angular. It was built with testing in mind, and having its own dependency injection system allows to mock absolutely everything. Angular team promoted testing heavily, so a lot of projects will have a lot of tests -- they will likely to be E2E, and run with karma, so it is not the fastest test suit, but I found it pretty straightforward and very helpful.

I guess you'll have some tests already, and you don't set up angular.js project from scratch (if you are doing it while reading this guide, you either know what you are doing or you should rethink your decision, at least in favour of Angular 2/4), so I'd just redirect you to several excellent resources how to test angular.js applications:  

- [Official docs]()
- []


## Minification

Angular has one limitation, due to its desire to parse everything. If you want, you can set parameters for your controller/factory functions other services/factories, as long their names are the same as they were registered. Angular parses this list, and decides what to inject; it works perfectly during the development, but all of the sudden fails after minification. The reason for that is that minifier has no idea about this behaviour, and just renames them to something like `a`, `b` (shorter the better!), and parsing stops to work. In order to solve this problem, array notation is used -- when we first list all dependencies names, and only then declare our function. Names should be 100% the same as we declare them (first argument in `directive`, `controller`, `factory`, etc).

- [Minification in official docs]()
- [How to minify angular.js code on SO]()

## Conclusion

Of course, I had to left behind a lot of other useful stull. But again, my goal was not to give a comprehensive guide to the whole Angular.js framework, rather just give understanding how it works and what are the most important blocks of it, which you will encounter while reading existing code (I don't think somebody will start a serious project using Angular.js, given that it is not going to evolve anymore). Angular.js has really bad reputation amongst JavaScript developers, but it is not that bad -- in case you need to support Angular.js application, think about it more pragmatically: it all made sense at the moment, 4-5 years ago Angular was the same as React nowadays, de-facto standard for all new shiny client-side applications.
