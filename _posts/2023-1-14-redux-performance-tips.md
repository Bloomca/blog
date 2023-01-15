---
layout: post
title: Redux Performance Tips
keywords: redux, performance, react, javascript, react-redux, react performance, performance tips
excerpt: "Redux Performance Tips when you really need to squeeze extra performance out of your React app"
---

[Redux](https://redux.js.org/) is a great state management library which allows to centralize all your data in one place and subscribe to it in individual components. Technically, it can be used with a pretty much any library (or even without) by subscribing to the store where needed, but usually it is used with React and I will focus on that combination in this article. I'll also use only functional components, I think they are the standard way of building React apps today.

First thing first, you might never need to think of the Redux performance. Unless you have a lot of data and components with complicated logic which often alter representation of that data, it rarely is a problem. But sometimes your app is big, and you want to squeeze all that performance, and it's good to know all potential issues.

## useSelector hook

First and the easiest thing to fix is to make sure you use [useSelector hook](https://react-redux.js.org/api/hooks#useselector) correctly. It executes the passed callback every time Redux store changes and if result is different from the last time, the whole React component re-renders. Usually it is exactly what you want, but you can accidentally return a new object or an array and that would mean a re-render every time your store changes.

{% highlight jsx linenos=table %}
function Component() {
    // will re-render on every Redux store change
    const { user, tasks } = useSelector(state => ({
        user: selectUser(state),
        tasks: selectTasks(state)
    }))
}
{% endhighlight %}

To solve it, usually the easiest way is simply to decouple selectors into individual hooks:

{% highlight jsx linenos=table %}
const user = useSelector(selectUser)
const tasks = useSelector(selectTasks)
{% endhighlight %}

If you need to construct an array, you can either memoize it using [reselect](https://github.com/reduxjs/reselect), or write a custom compare function, more on that later.

## Primitives are always memoized

The default compare function is just referential equality, and for [primitive types](https://developer.mozilla.org/en-US/docs/Glossary/Primitive) it means they will be just compared directly. This means that returning strings, numbers or booleans is always preferred, and you should deduct these values immediately inside the selector. Consider the following example:

{% highlight jsx linenos=table %}
const comments = useSelector(state => selectComments(state))
const commentsNumber = comments.length
{% endhighlight %}

While `selectComments` can be memoized properly and overall work just fine, in case we update comments in real time (e.g. somebody edited a comment), the component where we use this data can re-render quite often on popular pages. If we need that data high enough in the tree, that can potentially cause performance issues. To fix that, we can simply access `length` property immediately in the `useSelector`, because it is a number it won't cause a re-render in case total number hasn't changed:

{% highlight jsx linenos=table %}
const commentsNumber = useSelector(state => selectComments(state).length)
{% endhighlight %}

## Custom useSelector compare function

Sometimes you need to return an object from the `useSelector` hook. Maybe the calculation you do is too expensive and you can't memoize it fully, or there are simply too many properties, or you need to build an array on the fly with the variable length. I know I advised against doing it (usually you can just do it by other means), but you can provide your own compare function. By default it is a referential equality function (regular `===` check), but we can do anything we want. Remember that the check is not free, so just adding [\_.isEqual()](https://lodash.com/docs/4.17.15#isEqual) everywhere is not the best idea.

To give you an idea, we can get comment ids and check if they are still the same:

{% highlight jsx linenos=table %}
const commentIds = useSelector(
    state => selectComments(state).map(comment => comment.id),
    (prev, next) => prev.length === next.length && next.every((comment, i) => {
        return comment === prev[i]
    })
)
{% endhighlight %}

This way the component will re-render only when comment IDs change, e.g. we get more comments or their order changes.

## Selecting an object

If you need to access data from the store, like a name from the user, it is a common practice to write a selector which will return the entire user object. It makes sense, it is reusable, and usually user object doesn't change that much.

{% highlight jsx linenos=table %}
function UserComponent() {
    const user = useSelector(selectUser)
    return (
        <div>
            <img src={user.img} alt="">
            {user.name}
        </div>
    )
}
{% endhighlight %}

To illustrate the problem, though, let's say that user object stores some preferences. Let's say settings open in a modal, and that user component will still be rendered in the background, and when you update these preferences, your component will re-render. Not ideal, but also not a big deal, because UserComponent is small, and there is only one user.

Now what if the user object has a `token` property, and you perform a request somewhere at the top of your virtual DOM? Now everytime the user object changes, if the component is big enough, we'll re-render the entire tree, which can cause a feeling that the app doesn't work smoothly. A potential fix is to either return only a token (which is probably a string and won't change), or, if you need several values from the user, write a custom compare function.

{% highlight jsx linenos=table %}
const token = useSelector(state => selectUser(state).token)
{% endhighlight %}

## Array of objects

Let's say that you have a Redux store with a `tasks` reducer, which just stores them all together in an object, indexed by ID. We have a properly memoized selector, which gets them all. There is a component which renders them all. Now, let's say that the task #31 changed its description. What happens next? Well, in a typical implementation the whole list will re-render. As I said before, in the vast majority of situations that's okay, tasks don't really change their descriptions that often. But it all changes if our data is interactive. If the _user_ can change it, we are facing many re-renders; people want snappy editing experience, and in case of user editing many tasks in a row the application might get not that responsive.

I have to say that there is no ideal solution, it's all about compromises. First, you really need to look how bad it is. If you employ pagination or virtual scrolling, the amount of elements can be manageable, and re-rendering the list won't be a big problem. Same with the maximum data size, maybe you have some limits which make it not the biggest offender. But if you still find yourself looking at a similar problem, there are some solutions.

For example, we can pass only IDs instead of entire objects. When a task changes, IDs won't change, and only the component which renders the changed task will update because the selector function will return a new object for it. This is not always possible, but remember that you can write a complex `useSelector` function, which will gather all needed data and return an object, for which you can write a custom compare function. To give you a brief example:

{% highlight jsx linenos=table %}
const { taskIds, hasSharedTasks, viewOptions } = useSelector(
    state => {
        const tasks = selectTasks(state)
        const hasSharedTasks = tasks.some(isTaskShared)
        const viewOptions = selectViewOptions(state)
        const sortedTasks = applyViewOptions(tasks, viewOptions)

        return {
            taskIds: sortedTasks.map(task => task.id),
            hasSharedTasks,
            viewOptions
        }
    },
    (prev, next) => prev.hasSharedTasks === next.hasSharedTasks &&
                    prev.viewOptions === next.viewOptions &&
                    compareArrays(prev.taskIds, next.taskIds))
)

function compareArrays(firstArr, secondArr) {
    if (firstArr.length !== secondArr.length) return false
    return firstArr.every((item, i) => item === secondArr[i])
}
{% endhighlight %}

This way we pass correctly sorted IDs, and individual tasks will re-render, without the whole list.

## Optimistic updates

Optimistic updates are great, they give immediate feedback to the user, but they have 2 potential re-rendering problems. First is the moment of the optimistic update. Usually we dispatch an action, a reducer runs and returns a new store, and our components update. The problem is related to the previous point, where I showed that in case of a big list, it can have some performance implications.

However, be aware that often there are 2 re-renders. While the first happens immediately, the second usually occurs when we receive a response from the server, because often we just replace a local object with the one coming from the backend. Often we can't do much about it, as there might be some metadata changes (like `updated_at` field, etc), but if you can do it, a deep comparison between objects can help.


## Lots of data

Sometimes you have a lot of data. Like a lot. And at that points selectors stop being free on slower devices. Memoization definitely helps, but it often is limited if you need several parameters for your selector, you'll often have to do full calculation at the first render of the component. To avoid that, consider denormalization: it has higher maintainence cost, but will help you to speed your application performance.

{% highlight ts linenos=table %}
// instead of flat structure:
type State = {
    tasks: {
        [id: string]: Task
    }
}

// consider denormalizing it a bit
type State = {
    tasks: {
        byId: {
            [id: string]: Task
        }
        byProjectId: {
            [projectId: string]: Task['id'][]
        }
    }
}
{% endhighlight %}
