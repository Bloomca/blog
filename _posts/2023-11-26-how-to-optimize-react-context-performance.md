---
layout: post
title: How to Optimize React Context Performance
keywords: performance, react, javascript, react context, react performance, performance tips, zustand
excerpt: "Avoid using React Context if you need to store and update more than 1 property, and try to split all Context data into individual Providers"
---

[React Context](https://react.dev/reference/react/createContext) is a great API to avoid prop-drilling or if you need a simple way to have access to some global app data in any component, but don’t want to use often recommended solutions like [Redux](https://redux.js.org/) or [Mobx](https://mobx.js.org/README.html), because they, being good scalable libraries, advocate for proper architecture and to separate all files properly, and that is a big daunting task for something small.

There are several canonical examples on how to use Context on a global scale. I would say people often give examples with the following things:

- theming
- authentication
- i18n (Internationalization)

The common pattern here is that these things rarely change, and when they do, we probably want to reload the whole application anyway, maybe except theme. So if your use-cases are indeed something similar, something which probably will not be changed and just needed to be read, then go ahead and use React Context. Realistically, we don’t update the user property very often, or we don’t update auth status very often as well, and so on.

> Please note that this article talks specifically about storing "global" values in Context, which can be accessed from any part of the app. React Context is a very good approach to build component libraries with more untuitive/declarative APIs, and the performance issues there are usually negligible (but that is not always the case), and in this case I'd advocate against premature optimizations.

## Overview

So, what's the big deal? Well, the issue is that it is relatively simple to introduce performance problems which can bite you much later, and you might never even know about them, since we usually develop on pretty powerful machines. Typically, the general advice is to avoid premature optimizations at all costs, but in this case it is just a good developer practice, from my experience. You are basically protecting yourself from performance regressions in the future. There is [an excellent article](https://www.developerway.com/posts/how-to-write-performant-react-apps-with-context) which describes in detail how you can improve your Contexts from Nadia Makarevich, which I highly recommend to read as well.

While it is possible to store [primitive values](https://developer.mozilla.org/en-US/docs/Glossary/Primitive) in Context and to separate value and setters, usually people opt for keeping them in the same Context, so we end up storing an object.

{% highlight js linenos=table %}
import { createContext, useState } from 'react'

export const SyncUpdatesContext = createContext({
    lastUpdate: null,
    setLastUpdate: () => {
        // noop
    }
})

export function SyncUpdatesContextProvider({ children }) {
    const [lastUpdate, setLastUpdate] = useState(new Date())
    const value = { lastUpdate, setLastUpdate }
    return (
        <SyncUpdatesContext.Provider value={value}>
            {children}
        </SyncUpdatesContext.Provider>
    )
}
{% endhighlight %}

The potential issue is that some components might only need to have access to the setter function, but does not need to know about the current value. For example, we might have a button which does not display the last update time, but can set the new update time. In general, this is a very neglible thing, as re-rendering one button won't hurt the performance much. The problem usually manifests when these components are rendered in a list: imagine a list where every item has this button, and maybe the button displays something else which makes the re-rendering more expensive.

In the mentioned examples before (for themes, auth and i18n), usually it is not a problem because the values rarely change, and even if they do, it often warrants an entire app refresh. It is also a solvable problem, we can just create 2 Contexts, and that will solve this specific issue:

{% highlight js linenos=table %}
import { createContext, useState } from 'react'

export const SyncLastUpdateContext = createContext(null)
export const SyncLastUpdateAPIContext = createContext(() => {
    // noop
})

export function SyncUpdatesContextProvider({ children }) {
    const [lastUpdate, setLastUpdate] = useState(new Date())
    return (
        <SyncLastUpdateContext.Provider value={lastUpdate}>
            <SyncLastUpdateAPIContext.Provider value={setLastUpdate}>
                {children}
            </SyncLastUpdateAPIContext.Provider>
        </SyncLastUpdateContext.Provider>
    )
}
{% endhighlight %}

## Several values

Sooner or later, we'd want to pack more values into some Context if we use them for some application-wide data. So even if we separate values and setters for them, we encounter a new issue: some components care only for one value, but not for another. This is a solvable problem as well, because we can just keep splitting each individual data into its own Context Provider. This will introduce a lot of layers, but for the consumer it doesn't matter, as we can wrap all the access APIs into their own hooks.

{% highlight js linenos=table %}
import { createContext, useState } from 'react'

const noop = () => {}
export const SyncUpdatesContext = createContext({
    lastUpdate: null,
    updatingStatus: 'idle',
    error: null,
    setLastUpdate: noop,
    setUpdatingStatus: noop,
    setError: noop
})

export function SyncUpdatesContextProvider({ children }) {
    const [lastUpdate, setLastUpdate] = useState(new Date())
    const [updatingStatus, setUpdatingStatus] = useState('idle')
    const [error, setError] = useState(null)
    const value = {
        lastUpdate, setLastUpdate, updatingStatus,
        setUpdatingStatus, error, setError
    }
    return (
        <SyncUpdatesContext.Provider value={value}>
            {children}
        </SyncUpdatesContext.Provider>
    )
}
{% endhighlight %}

It makes sense to bundle all these properties together, because they all correspond to the same thing, but you can clearly see where it is going. With more and more properties, different components might subscribe to it and we'll end up updating more than we should. Similar to the previous example, we can just create own Contexts for each value, and all setters can be bundled to a single Context since they don't change, something like this:

{% highlight js linenos=table %}
import { createContext, useState, useMemo } from 'react'

const noop = () => {}
export const SyncLastUpdateContext = createContext(null)
export const SyncUpdatingStatusContext = createContext('idle')
export const SyncErrorContext = createContext(null)
export const SyncUpdateAPIContext = createContext({
    setLastUpdate: noop,
    setUpdatingStatus: noop,
    setError: noop
})

export function SyncUpdatesContextProvider({ children }) {
    const [lastUpdate, setLastUpdate] = useState(new Date())
    const [updatingStatus, setUpdatingStatus] = useState('idle')
    const [error, setError] = useState(null)
    const apiCommands = useMemo(() => ({
        setLastUpdate, setUpdatingStatus, setError
    }), [setLastUpdate, setUpdatingStatus, setError])
    return (
        <SyncLastUpdateContext.Provider value={lastUpdate}>
            <SyncUpdatingStatusContext.Provider value={updatingStatus}>
                <SyncErrorContext.Provider value={error}>
                    <SyncUpdateAPIContext.Provider value={apiCommands}>
                        {children}
                    </SyncUpdateAPIContext.Provider>
                </SyncErrorContext.Provider>
            </SyncUpdatingStatusContext.Provider>
        </SyncLastUpdateContext.Provider>
    )
}
{% endhighlight %}

This will solve the problem, and is a pretty good solution overall. I wouldn't prematurely optimize a Context like that, though, and only do that if there is a direct performance impact, usually it means you need some of these properties in every item of a potentially long list.

## Processing values

Finally, we reach the issue which is not really solvable with the Context on its own. What if we need the result of some calculation, and not the value itself? To illustrate that, let's say that we store an active selected item. Everything else can be separated into their own Contexts, so we don't need to store anything there. And then, in the app, we have a list of components, where each item needs to understand whether it is selected or not. A typical code will look like this:

{% highlight js linenos=table %}
import { createContext, useState } from 'react'

const SelectedItemContext = createContext(null)
const SelectedItemAPIContext = createContext(() => {})

function SelectedItemProvider({ children }) {
    const [selectedItemId, setSelectedItemId] = useState(null)
    return (
        <SelectedItemContext.Provider value={selectedItemId}>
            <SelectedItemAPIContext.Provider value={setSelectedItemId}>
                {children}
            </SelectedItemAPIContext.Provider>
        </SelectedItemContext.Provider>
    )
}

function ItemComponent({ item }) {
    const selectedItemId = useContext(SelectedItemContext)
    const isSelected = selectedItemId === item.id

    return (
        <div className={isSelected ? 'item__selected' : undefined}>
            {content}
        </div>
    )
}
{% endhighlight %}

When a new item is selected, all items will re-render, because `selectedItemId` has changed, even though the only thing we are interested in is whether `isSelected` changed. For the most items, the answer will be negative. Only 2 components need to re-render: the previously selected item, and the newly selected item. We can technically optimize the performance by passing down content as `children` from a component above, but that would work only if `isSelected` is not needed for any children, and sooner or later, it will be needed.

## Solution

Good news is that we can solve this problem as well. React provides API to update values without triggering state changes (and re-renders), [useRef](https://react.dev/reference/react/useRef). It is a stable object reference, which we can update, but can not subscribe for changes. We can utilize to provide smarter way to subscribe to changes.

Let's write some code:

{% highlight js linenos=table %}
import { useRef, createContext } from 'react'

export const SelectedItemContext = createContext({ current: null })

function createSelectedItemStore() {
    const listeners = []
    let state = {
        selectedItemId: null,
    }

    return {
        getState: () => state,
        setState: (newState) => {
            state = newState
            listeners.forEach((cb) => cb(state))
        },
        subscribe: (cb) => {
            listeners.push(cb)

            // remove the listeners for the callback
            return () => {
                const cbPosition = listeners.findIndex(cb)
                listeners.splice(cbPosition, 1)
            }
        },
    }
}

export function SelectedItemContextProvider({ children }) {
    const stableValue = useRef(createSelectedItemStore())
    return (
        <SelectedItemContext.Provider value={stableValue}>{children}</SelectedItemContext.Provider>
    )
}
{% endhighlight %}

Now we need a way to access these values, but without triggering unnecessary updates. The way to do is to add support for selectors, so that this way the state update will trigger only if the end result is different.

{% highlight js linenos=table %}
import { useContext, useEffect, useState } from 'react'

export function useSelectedItem(cb) {
    const selectedItemContextStore = useContext(SelectedItemContext)
    // we need to calculate the value immediately
    const [value, setValue] = useState(() => cb(selectedItemContextStore.current.getState()))
    // subscribe function should not update unnecessarily, so we pass all values through stable refs
    const refValue = useRef(value)
    const cbRefValue = useRef(cb)

    const unsubscribe = useMemo(
        () =>
            selectedItemContextStore.current.subscribe((state) => {
                const newValue = cbRefValue.current(state)

                if (refValue.current !== newValue) {
                    refValue.current = newValue
                    setValue(newValue)
                }
            }),
        [selectedItemContextStore],
    )

    // update callback function if it changes
    useEffect(() => {
        cbRefValue.current = cb
    }, [cb])

    useEffect(() => unsubscribe, [unsubscribe])

    return value
}

export function useSetSelectedItem() {
    const selectedItemContextStore = useContext(SelectedItemContext)

    return (id) => selectedItemContextStore.current.setState({ selectedItemId: id })
}
{% endhighlight %}

Finally, we can use it in our components the following way:

{% highlight js linenos=table %}
export function Item({ item }) {
    const isSelected = useSelectedItem((state) => state.selectedItemId === item.id)
    const setSelected = useSetSelectedItem()
    return (
        <div className={`item ${isSelected ? 'item__selected' : ''}`}>
            <h3>{item.content}</h3>
            <button onClick={() => setSelected(item.id)}>Make selected</button>
        </div>
    )
}
{% endhighlight %}

Now we have the optimized version where only 2 items will re-render, for whom `isSelected` changed. The best part about this solution is that it doesn't have previous drawbacks: we can store as many properties as possible, and while I just exported a general `setState` function, we can totally provide as many granular setter functions as we need. We still utilize React Context to store it, but we do not rely on it anymore to track updates, each consumer hook will take care of that.

## Note on React Context

Technically, we don't need to use React Context for the store at all. If you use it for global values, you can just create it in memory and access through the hooks only, there is basically no downside to that approach, but there are two potential issues:

- you can't use it on the server, because you need to create a new store for every user
- sometimes you need to have separate Contexts for certain parts of the application

If neither of those apply, feel free to skip storing it in the Context Provider.

## You Don't Need to Do That

The thing about the code above is that it is not really unique. Pretty much every React store implements something like that, and it is not a coincidence that the API resembles Redux a lot. So that means that you don't need to write that part by yourself (and to be honest, my implementation is very basic, no middleware/logging/etc), there are plenty of libraries to do so for you. I've used [Zustand](https://github.com/pmndrs/zustand) and can recommend it highly. It is a very small ([1.17kb bundled](https://bundlephobia.com/package/zustand@4.4.6)) library, can even be used instead of Redux for smaller-scale apps, and can be used instead of React Context. Here is what they list:

> Why zustand over context?
> - Less boilerplate
> - Renders components only on changes
> - Centralized, action-based state management

The idea is relatively simple, so I am sure there are other libraries with the same idea: to make more robust applications in React. So for the price of 1kb (or even less if you write your own) you can create performance-protected more easily extendable global stores.
