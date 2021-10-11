---
layout: post
title: Performance Optimization in React applications
keywords: react, performance, hooks performance, redux performance, selector performance
---

Let's talk about React performance. React is a great framework that makes a contract with us: we feed it data, and it renders us the content. When we subscribe to our services and update the internal state of React components, it guarantees that everything will be up-to-date on the screen. Not only that, but it is also efficient at swapping DOM nodes: after rendering it compares the result tree to the current DOM and swaps what is needed.

In theory, it sounds like we can just subscribe to any changes and not worry about performance at all: the framework will sort it out and will make it efficient enough, right? Well, unfortunately, no. It might work just fine being not very optimized if your application is fairly small, but as soon as your components start to get a lot of calculations inside their render functions and a lot of children, even calculating that tree will become quite an operation, not to mention comparing it to the current DOM tree. Besides, inefficient apps are not great for battery life and temperature, as they use the CPU more aggressively.

## Table of Contents

- [Reasons for re-rendering](#reasons-for-re-rendering)
- [Memoization](#memoization)
- [Inefficient props drilling](#inefficient-props-drilling)
- [React.useState hook](#reactusestate-hook)
- [React.useMemo/useCallback hooks](#reactusememousecallback-hooks)
- [React Context](#react-context)
- [Redux selectors](#redux-selectors)
- [Redux selectors with parameters](#redux-selectors-with-parameters)
- [Fundamental problem with Redux data](#fundamental-problem-with-redux-data)
- [Conclusion](#conclusion)


## Reasons for re-rendering

React components re-render for 2 reasons: either the internal state of the component changes or its parent changed its state. The second part is especially tricky because it has a cascading effect: a changed state will cause all their children to re-render. It is also true for stopping re-rendering, if one component cuts re-rendering shortly through memoization, all their children also will not be re-rendered.

It means that you have to be especially careful when subscribing to changes in top-level components, and if you absolutely have to do it, memoize components where these changes don't affect them. When you memoize, you have to be extremely careful about passed properties, because it is very convenient to create new objects/functions inline in properties, but that can backfire; we'll talk more about it in [React.useMemo/useCallback hooks](#reactusememousecallback-hooks) section.

> Note: I will use only function components in this article. They are more composable than class-based components, allow to reuse logic via hooks, and in general, I think are considered to be the preferred approach in the community.

<p class="centred-image full-image">
  <img class="image" src="/assets/img/should-component-update.png" />
  <em>Diagram from the <a href="https://reactjs.org/docs/optimizing-performance.html#shouldcomponentupdate-in-action" target="_blank" rel="noreferrer">official docs</a></em>
</p>

You can read a general overview of the performance optimization [in the docs](https://reactjs.org/docs/optimizing-performance.html), and here we'll dive into more specific issues and how to solve them.

## Memoization

The first trick in the journey of optimization is usually to look at memoizing individual components. For example, imagine a situation where you render a list of comments: you usually have a container that iterates over all available comments and renders them individually. Now, let's say that we received a new comment, and now our list has changed. By default, the application will re-render the container and all its children, even though in reality we only need to re-render the container info which shows a number of comments and the new comment, while all older comments don't really need to change, as they are exactly the same.

This is the prime example of memoizing a component. To combat that situation, we wrap our component into a [React.memo](https://reactjs.org/docs/react-api.html#reactmemo) function, which will do the following: it will remember all passed props, and then next time new props are passed, they will be compared using "shallow comparison", which means it checks equality of objects' properties using referential equality, which means it uses `===` operator to compare them. If all are exactly the same, it won't even go inside the render; this will prevent all its children from re-rendering as well.

There are some caveats with using `React.memo`, though.

Do not just wrap your components in `React.memo`! If you pass a newly created object/function into them, or some property indeed changes every time, they will re-render anyway, but now before every render it will try to compare all the props, causing unnecessary object comparisons, which can even make performance slightly worse.
Also, if the component is re-rendered because of some internal state, memoization will not happen. In that case, though, there will be no overhead, but no benefit as well; you can memoize children components if they are expensive to calculate and don't change.

---

A note about [refs](https://reactjs.org/docs/refs-and-the-dom.html): even though they are not treated as direct properties, they are, and used in memoization check as so. So if you create a new function in the passed `ref` property, `React.memo` will not work at all and be in the same situation as described before.

## Inefficient props drilling

Sometimes properties indeed change, and we can't do much about it, aside from writing our own memoization function. `React.memo` accepts the second argument, where we can write our own comparison function and fine-tune whether we want the component to re-render or not. This is a special technique that can be useful, but it is also extremely dangerous and probably should be avoided in most cases. The reason for that is that you can cut off re-rendering early, but the problem might be that some children rely on that "ignored" property and you can get some special cases where the application is not updated properly.

To illustrate, let's have an example:

{% highlight jsx linenos=table %}
function Comment({ comment }) {
  return (
    <div>
      {comment.content}
      <Reactions comment={comment} />
    </div>
  );
}

function Reactions({ comment }) {
  return (
    <ul>
      {Object.entries(comment.reactions).map(([reaction, userIds]) => (
        <li key={reaction}>
          {reaction}
          {userIds.length}
        </li>
      ))}
    </ul>
  );
}

const MemoizedComment = React.memo(
  Comment,
  (prevProps, nextProps) => prevProps.content === nextProps.content
)
{% endhighlight %}

You probably can immediately see the issue here: the component will re-render only when the `content` property changes, while we clearly need to track `reactions` as well. In bigger components, it is much easier not to notice some property that is needed down the line, so if you really want to go down that road, I recommend passing properties individually, `content` and `reactions` as separate properties. Still, sometimes it is too involved and it is the only option, then go for it and be very careful about changing that component down the line.

## React.useState hook

Today React applications are mostly written using function components, and they handle internal state using [hooks](https://reactjs.org/docs/hooks-intro.html). They are convenient and allow for extracting and reusing state functionality, but they also introduce some potential performance issues.

[React.useState](https://reactjs.org/docs/hooks-reference.html#usestate) hook is used to store data between re-renders. It is pretty straightforward, but it is important to understand how it works when we set a new value. If we store a primitive value, we can safely set the new value there and if it is the same, the component will not re-render. But if we store an object, and when we set a new one, it will treat it as a forced re-render. So, when we do something like this:

{% highlight jsx linenos=table %}
const [data, setData] = React.useState({ value: '', expanded: false })

function handleClose() {
    setData(currentData => ({ ...currentData, expanded: false }))
}
{% endhighlight %}

If `handleClose` function is called, even if the state wasn't changed, it will re-render the component anyway. While you might want to check whether the new state is needed and return `currentData` object if nothing has changed, a much better solution would be to split that state into 2 separate values:

{% highlight jsx linenos=table %}
const [value, setValue] = React.useState('')
const [isExpanded, setIsExpanded] = React.useState(false)
{% endhighlight %}

This way you don't need to worry about potential unnecessary re-rendering. This principle is also relevant when creating your own hooks, or passing callbacks that return data in other hooks, like in [react-redux useSelector](https://react-redux.js.org/api/hooks#useselector).

For example, the hook below has the same issue as in the previous example:

{% highlight jsx linenos=table %}
function useOurHook() {
    const [onlineStatus, setOnlineStatus] = React.useState({
      online: true, backendOnline: true
    })

    React.useEffect(() => {
        const onlineListener = () => {
          setOnlineStatus(status => ({ ...status, online: true }))
        }
        const offlineListener = () => {
          setOnlineStatus(status => ({ ...status, online: false }))
        }

        document.addEventListener('offline', offlineListener)
        document.addEventListener('online', onlineListener)

        // to make sure that online status is always correct
        const interval = setInterval(() => {
            setOnlineStatus(status => ({
              ...status, online: navigator.onLine
            }))
        }, 1000)

        // This is just a generic interface we can provide
        const unsubscribe = backendStatus.subscribe((status) => {
            setOnlineStatus(status => ({ ...status, backendOnline: status }))
        })

        return () => {
            document.removeEventListener('offline', offlineListener)
            document.removeEventListener('online', onlineListener)
            unsubscribe()
            clearInterval(interval)
        }
    })

    return {
      isOnline: onlineStatus.online,
      isBackendOnline: onlineStatus.backendOnline
    }
}
{% endhighlight %}

This example is a bit exaggerated because we don't technically need `setInterval`, but there might be some bug on our supported platforms that `online` events are not always triggered correctly, so it is possible to end up with something similar.

When you look at it, this example seems to be a fine hook, which should change only when `isOnline` or `isBackendOnline` change. Despite never changing property values, `onlineStatus` will refer to the new object every second. The fix is the same as before: split one state into individual states which will track a single boolean value. If we don't do that, every component which uses this hook will re-render unnecessarily.

## React.useMemo/useCallback hooks

Hooks can also help with building more efficient applications. Sometimes we need to modify data before passing it to children, and doing that will cause the child, even wrapped in `React.memo`, to re-render anyway. Let's look at the example:

{% highlight jsx linenos=table %}
function CommentsContainer({ comments, userId }) {
    const userComments = comments.filter(
      comment => comment.author === userId
    )

    return (
        <main>
            <Summary comments={userComments} />
        </main>
    )
}
{% endhighlight %}

In this example, we create a new array `userComments` every time the component is rendered, and that would prevent any memoization in the `<Summary />` component. While usually components are not memoized and it is not a problem if we render a memoized component we need to be extra careful what we pass into it. While we can solve this problem with the combination of `React.useState` and [React.useEffect](https://reactjs.org/docs/hooks-reference.html#useeffect) hooks to update userComments every time `comments` property changes, there is a hook which can do that for us: [React.useMemo](https://reactjs.org/docs/hooks-reference.html#usememo). It sounds familiar to `React.memo`, but it works inside components. To show how we can improve the previous example:

{% highlight jsx linenos=table %}
function CommentsContainer({ comments, userId }) {
    const userComments = React.useMemo(() =>
      comments.filter(comment => comment.author === userId),
      [comments]
    )

    return (
        <main>
            <Summary comments={userComments} />
        </main>
    )
}
{% endhighlight %}

Now, `userComments` will receive a new array only if `comments` property has changed. It is still not ideal, because even if `comments` changed, it does not mean that `userComments` did change, but it would involve a lot of changes to solve that issue, while typically memoizing is enough.

The same idea is true for [React.useCallback](https://reactjs.org/docs/hooks-reference.html#usecallback). In fact, these two expressions are equivalent:

{% highlight jsx linenos=table %}
// example of onChange handler we might want to pass down
React.useMemo(() => (newValue) => { setValue(newValue) }, [])
React.useCallback((newValue) => { setValue(newValue) }, [])
{% endhighlight %}

---

One thing you might be tempted to do is to wrap everything in `useMemo/useCallback`. It is not really needed, because if the component receives new properties, it means that the parent is re-rendering anyway. Often we are okay with rendering children with the parent, but in case you optimize some expensive components, carefully look at the properties you pass and make sure they are not newly created objects on each render.

## React Context

[React Context](https://reactjs.org/docs/context.html) is a great way to store data in specific components tree, but you have to be aware that at the moment it is not super-efficient. There is no way to directly subscribe to specific data in context, so every time you change something in it, all subscribed components will re-render, including their children. Because you have to pass a function to change context data as well, you can't store a primitive value, so it will always end up a new object when you change some data inside, which will cause all children, which are subscribed to the context, to re-render. To make sure that it doesn't bring any issues, try not to create big contexts with several properties which are not really related to each other, and ideally split them into separate contexts or store them in your store, like [Redux](https://redux.js.org/) or [Mobx](https://mobx.js.org/README.html).

If you can't do that, or you have a very specific use-case where these approaches won't work for you, create a wrapper component that will pass down an individual property and memoize the component itself:

{% highlight jsx linenos=table %}
const MemoizedComponent = React.memo(Component)

function WrapperComponent(props) {
    const { value } = useMyContext()

    return <MemoizedComponent value={value} ...props />
}
{% endhighlight %}

This works with custom hooks as well: pass down only relevant properties, and memoized component will make sure that it will re-render only when something changes, not just when the whole context/hook is changed.

## Redux selectors

I'll focus on `Redux` selectors, but generally speaking, it is applicable to any data querying. Let's say that we have some normalized data, and we want to gather all comments. Internally, we store them as a dictionary of all comments mapped by ID, and then map IDs to threads. This simplifies many things, like editing comments, but it means that we often need to collect data before using it.

Consider this data structure:

{% highlight jsx linenos=table %}
const state = {
    comments: {
        byId: {
            [1]: {...},
            [2]: {...},
            [3]: {...},
            [4]: {...},
            [5]: {...},
        },
        byThread: {
            [1]: [1, 2],
            [2]: [3, 4, 5]
        }
    },
    view: {
        threadId: 2
    }
}
{% endhighlight %}

If we need to get all comments in a thread, we can either get a list of IDs and then get individual objects inside `<Comment />` component, or we can get a list of all comments where IDs are replaced with comment objects. The second approach is more convenient because you can directly use comments objects, and often just a list of IDs is not enough. Let's write a simple function to get a list of the thread's comments:

{% highlight jsx linenos=table %}
// this is a simplified version which assumes data is always there
function selectThreadComments(state) {
    return state.comments.byThread[state.view.threadId].map(
      id => state.comments.byId[id]
    )
}
{% endhighlight %}

The problem is that every time we call this function, we construct a new array, and that forces our [useSelector](https://react-redux.js.org/api/hooks#useselector) hook to cause the component's re-render. To combat that, we need to make sure that if the state didn't change, we will return exactly the same result as before. To be precise, we need to make sure that if _relevant_ state didn't change, we won't recalculate the selector and return the same array. There is already a great library [reselect](https://github.com/reduxjs/reselect), which memoizes all your state computations, and then, if they are the same, returns the memoized result. Our function will be rewritten to:

{% highlight jsx linenos=table %}
import { createSelector } from 'reselect'

const selectThreadComments = createSelector(
    state => state.comments.byThread[state.view.threadId],
    state => state.comments.byId,
    (threadCommentIds, commentsById) => threadCommentIds.map(
      id => commentsById[id]
    )
)
{% endhighlight %}

And as long as the state doesn't change the outcome of the first 2 functions, where we get `threadCommentIds` and `commentsById`, the third function won't be executed and just return the previous result. You can fine-tune the comparator function, but in my experience, it is extremely good and if your selector depends only on the state, there is little reason not to use `createSelector`.

## Redux selectors with parameters

Selectors often need additional data than just the state. Sometimes we can get all the needed data from other stores, like in the example above, but often we need to use some additional parameter to correctly select needed data. This is a tricky topic because often you'll need to tweak things to make them work for your specific situation, but it is possible to optimize these selectors too. There is a [section](https://github.com/reduxjs/reselect#q-how-do-i-create-a-selector-that-takes-an-argument) in official docs of `reselect` library, and they offer either to move the parameter into the state or memoize the return function. Let's write down their example here:

{% highlight jsx linenos=table %}
const createExpensiveSelector = createSelector(
  state => state.items,
  items => memoize(
    minValue => items.filter(item => item.value > minValue)
  )
)

function FirstComponent() {
    const expensiveFilter = useSelector(createExpensiveSelector)
    const value = expensiveFilter(10)
}

function SecondComponent() {
    const expensiveFilter = useSelector(createExpensiveSelector)
    const value = expensiveFilter(10)
}

function ThirdComponent() {
    const expensiveFilter = useSelector(createExpensiveSelector)
    const value = expensiveFilter(100)
}
{% endhighlight %}

Let's look at [memoize](https://lodash.com/docs/4.17.15#memoize) function first. It remembers inputs, and if they all are the same as the previous call (again, using `===` operator), it skips the function execution and just returns the previous calculation result. When inputs are different, it doesn't just forget the previous result, it maps them to the first argument as a key by default. This is a bit of a dangerous behaviour if you pass more than 1 argument, so be careful when using this function, but it allows excellent performance due to effective memoization.

Let's get back to our example and how it works: we have 3 components that use the same selector. Since `state` will be the same, all 3 components will receive the same function as the result of `useSelector(createExpensiveSelector)` (and as soon as `state` changes, they will all receive a new memoized function, but it will be the same for all of them, because the first call will create that function and subsequent calls will return it).

Using this strategy you can efficiently use selectors which need additional parameters. You can also flip the `memoize` function usage and wrap the whole function with it, this way you can use it in selectors:

{% highlight jsx linenos=table %}
const createExpensiveSelector = memoize((minValue) {
    return createSelector(
      // we can use arguments here
      state => state.items,
      items => items.filter(item => item.value > minValue)
    )
})
{% endhighlight %}

## Fundamental problem with Redux data

This point is partially related to [Inefficient props drilling](#inefficient-props-drilling) and while it can be avoided, sometimes it is extremely daunting and would require to pass a lot of properties individually, which introduces its own set of problems.

The problem is that sometimes the data legitimately changes, but our component is not interested in that specific change. To illustrate, let's look at our `comment` data:

{% highlight ts linenos=table %}
type Comment = {
  id: number
  created_ts: number
  edited_ts: number
  content: string
  // map of reactions to user IDs
  reactions: { [reaction: string]: number[] }
}
{% endhighlight %}

Now, let's say that we changed `edited_ts` property, and that caused our component which renders reactions to re-render itself. Technically, it needs only `id` and `reactions` properties, and while it is possible to pass them directly, it is not convenient: if we need another property, it might require a lot of changes while refactoring, or if we select the comment data in the component directly, we'd need to either have 2 selectors to select properties individually or create a selector which will memoize the newly created object. I am specifically mentioning the `Redux` here because often you need to select some object by ID, and it is not feasible to use `useSelector` hook 5 times to get each property individually for the sake of performance.

I am not aware of any universal solution for this specific issue. The problem is that it takes time to realize that components are re-rendered inefficiently, and if we try to optimize heavily, it removes some flexibility; there is no one answer, but if you are absolutely stuck with performance and need to optimize it further, definitely look into passing individual properties instead of complete objects. If the property comes from the parent, you can write a custom comparison function for `React.memo`, but it has the issue that if you start to use a new property, you'll need to manually add it to your comparison function.

## Conclusion

These are just issues I've encountered when I was improving performance in a couple of big applications. There are more patterns and potential issues, but these are probably the biggest offenders. When optimizing, remember that premature optimization is not the best idea, and always measure when applying these changes!