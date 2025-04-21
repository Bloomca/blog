---
layout: post
title: Problem with React Update Model
keywords: react, components, UI library, reactivity
excerpt: "React's update model based on component-level updates is not the most efficient."
---

React is a great library, and its core principle redefined how we approach UI applications on the Web: declarative components with a single `render` method that create exactly the same output as long as the state is the same (and it encouraged putting all side-effects into the local state).

Almost every library since then follows this model of reusable declarative components.

## The problem

The issue is specific to React but can't be ignored as an implementation detail, because it would require a change in how state is handled. React's update model is tied to VDOM, which builds the internal tree of the component's markup, which is then compared to the current version. The comparison algorithm is extensive and efficient, and that's good enough for small to medium applications.

Once you have a bigger app, or just really inefficient components, you will run into problems. There are special [helpers](https://react.dev/reference/react/memo#skipping-re-rendering-when-props-are-unchanged) to address that, but they can only do so much.

In short, the main issue is that React is not great at reactivity (ironically): if you have a component depending on multiple states, any state change will re-render the entire component, including situations where the state is not used in the resulting DOM directly. There are techniques to avoid that, and often they are enough, but the problem is that it's brittle: it's not easy to fix and very trivial to break again, often without even realizing that you're introducing a regression.

## useEffect

[useEffect](https://react.dev/reference/react/useEffect) allows us to react to changes and is often used to run side-effects. The problem is that logically, a component might be responsible for running side effects but not use the required data outside of it. So you end up listening and re-rendering your component (which amounts to nothing since React will compare the current and new versions of internal structure; they will be the same, so no DOM changes will happen) even though only the side effect needs the changed values.

Consider this example:

{% highlight js linenos=table %}
import { useEffect } from 'react'
import { useData } from '../hooks'

function SomeComponent({ prop1, prop2 }) {
    const data = useData(data => data.someProp)

    useEffect(() => {
        doSomething(prop2, data)
    }, [prop2, data])

    return <div>{prop1}</div>
}
{% endhighlight %}

This component will re-render if:

- `prop1` changes
- `prop2` changes
- `data` changes

> Note: technically, `useData` might be inefficient and cause more re-renders, but we'll assume it's optimized

Yet the only value used in the actual markup is `prop1`. We can fix it by extracting the effect into a separate component:

{% highlight js linenos=table %}
function SomeComponentEffects({ prop2 }) {
    const data = useData(data => data.someProp)

    useEffect(() => {
        doSomething(prop2, data)
    }, [prop2, data])

    return null
}

function SomeComponent({ prop1 }) {
    return <div>{prop1}</div>
}

function parentComponent() {
    return (
        <>
            <SomeComponentEffects prop2={prop2} />
            <SomeComponent prop1={prop1} />
        </>
    )
}
{% endhighlight %}

The problem here is that by default, we won't do that; we'll co-locate the effects next to the markup.

## Data in callbacks

A similar issue is that some data isn't used in the markup directly but is used in callbacks like event listeners. This case is logically even worse than the previously mentioned side-effects: when the data changes, absolutely nothing needs to run because the value is only relevant when the callback is executed. Consider this example:

{% highlight js linenos=table %}
import { useData } from '../hooks'

function SomeComponent({ prop1, onClick }) {
    const isActive = useData(state => state.isActive)
    return <div onClick={() => {
        if (!isActive) return

        onClick(prop1)
    }}>{prop1}</div>
}
{% endhighlight %}

The component will re-render every time `isActive` changes, even though it's irrelevant for the component's markup. This can be partially solved: for example, you can use [useStore](https://react-redux.js.org/api/hooks#usestore) from `react-redux` to directly access store in the callback itself, a similar technique can be used for [Zustand](https://github.com/pmndrs/zustand) stores, but:

- it's a bit confusing
- you won't always have access to this method (specifically, [React Context](https://react.dev/learn/passing-data-deeply-with-context) cannot be used like that)

## Attributes

One thing that doesn't translate super well into DOM and entire component updates is DOM attributes. Usually it's not a problem because updates in the attributes are directly tied to the markup update. For example, a button changes its `disabled` attribute, so the actual DOM gets re-rendered.

However, sometimes you need to update attributes independently of the markup. For example, you might annotate your app using [data attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/data-*) to show that the item is available to be dragged or dropped. This is also a solvable problem: if you extract the wrapper with the changing attribute and pass children via the `children` prop, they will re-render based on the parent updates.

For example:

{% highlight js linenos=table %}
import { useDnD, isDroppingAllowed } from '../use-dnd'

function MyComponent({ item }) {
    const isAllowed = useDnD(dnd => isDroppingAllowed(dnd, item))

    return (
        <div data-is-drop-allowed={isAllowed}>
            {item.content}
        </div>
    )
}
{% endhighlight %}

The content might be a pretty complicated component with its own logic. You can see that we can extract the outer `<div>` in a clean manner, ending up with:

{% highlight js linenos=table %}
import { useDnD, isDroppingAllowed } from '../use-dnd'

function MyComponentDnDWrapper({ item, children }) {
    const isAllowed = useDnD(dnd => isDroppingAllowed(dnd, item))

    return (
        <div data-is-drop-allowed={isAllowed}>
            {children}
        </div>
    )
}

function ParentComponent({ item }) {
    return (
        <MyComponentDnDWrapper>
            <ItemContent item={item}>
        </MyComponentDnDWrapper>
    )
}
{% endhighlight %}

While this works, it's very rare that you'll have such a clean separation. Often you'll have some event listeners, classnames, etc., so you'll be splitting the logic between two components that are logically a single one. This technique can also be hard to implement in practice if you have multiple wrappers that are not top-level DOM elements in your component.

If you can't extract it, your entire component will re-render based on the external DnD state, even if your DnD implementation reads the values from the DOM/internal state directly and no re-renders are necessary.

## Multiple props

If you have multiple props (either passed or obtained from your hooks), your component will re-render each time any of them changes, even if you only use them together for some boolean condition. Usually it doesn't matter, but in the case of expensive components and complicated apps where you have multiple stores for different purposes, things will get complicated if you're trying to optimize. For example, this example is not really possible to optimize easily:

{% highlight js linenos=table %}
import { useStorage, useDnD, useFiltering } from '../hooks'

function MyComponentDnDWrapper({ content }) {
    const data = useStorage(state => state.data)
    const isDnDActive = useDnD(state => state.isActive)
    const isFiltering = useFiltering(state => state.isFiltering)

    const disabled = data && !isDnDActive && isFiltering

    return (
        <button disabled={disabled}>
            {content}
        </div>
    )
}
{% endhighlight %}

This example is pretty harmless; at the end of the day, it's just a button, and we pass content as a property, so it will only be re-rendered in the context of the parent. However, once you get components with complicated setups, this example might get much trickier. What's even worse is that it will very likely be hidden behind some hook, maybe even a dependency of a hook.

## Conclusion

In general, these issues are pretty well-known. There's a [React compiler](https://react.dev/learn/react-compiler) project, which will solve some of these "for free", and there are tons of articles and talks about how to understand and optimize React better. I highly recommend the [Advanced React](https://www.advanced-react.com/) book if you want to dive deeper. There are also more potential problems, but I think these are the biggest offenders.

The problem is that fundamentally, these issues will remain with us until the component definition, side effects, and actual DOM updates are completely decoupled from each other. It's hard to say what exactly the solution is, but I think libraries like [Solid.js](https://github.com/solidjs/solid) and my own [Veles](https://github.com/bloomca/veles) have a better approach to these specific problems (state updates are tied to specific DOM nodes and components are only executed once); overall, of course, React has an ecosystem which is extremely hard to match.

By themselves, each of these problems is solvable (outside of props from different sources), but when you get complicated components, "solving" starts to introduce real friction to the developer experience, and in some cases might be borderline impossible or not worth it due to the brittle nature of the fixes.