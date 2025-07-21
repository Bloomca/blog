---
layout: post
title: Multiple Windows in Electron apps
keywords: Electron.js, Electron, multi-window implementation, syncing between multiple windows
excerpt: "It is possible to provide multi-window/multi-instance experience in Electron, but it is probably not worth it."
---

A lot of desktop applications provide a multi-window experience. This can be done in 2 flavours:

1. Multiple instances of the same application (2 separate processes)
2. Multiple windows of the same application instance (one parent process)

I already mentioned in the [custom protocols article](/2025/07/20/electron-apps-custom-protocols#how-to-handle-deeplinks), the first approach is more native to Windows/Linux and the second approach is more native to macOS. Allowing to open multiple instances depends on what your application does and how independent each instance can be -- the more and faster you need to sync data between the processes, the less enticing that option becomes; plus allowing multiple windows will work the same across all platforms (while it is possible to open multiple instances of the same on macOS from the command line, it is definitely not a common approach).

The main advantage of requesting a [single instance lock](https://www.electronjs.org/docs/latest/api/app#apprequestsingleinstancelockadditionaldata) is that you get [inter-process communication (IPC)](https://www.electronjs.org/docs/latest/tutorial/ipc) _almost_ for free.

## Disabling multi-window

Depending on your application, there is nothing wrong with requesting a single instance lock and handling all deeplinks and additional instances events by activating and navigating in your main app window. This will dramatically reduce the complexity of syncing instances between each other (if necessary), eliminate a lot of potential edge cases and race conditions, and can provide a nicer flow to your application.

Multi-window makes the most sense for applications where you can have multiple independent documents/views; document-based applications are a great example. In something like [VS Code](https://code.visualstudio.com/) you can easily open multiple workspaces/folders and it is a natural requirement. But System Settings application can only have one active window/instance both on macOS and Windows, and it is 100% a sensible solution.

## Implementing multiple app instances

If you want to allow multiple application instances on Windows/Linux, I'd strongly recommend to not sync anything in real time. You can check saved settings files from time to time when it is a good moment (like check workspace settings when navigating to that workspace) or something similar; since it is the same application, it will have access to the same folders.

Otherwise you'll build something brittle which might be expensive to maintain, IPC is a very complex problem. Besides, on macOS you'd need to support multiple windows for parity, and you can simply reuse the same solution by requesting a single lock.

## Implementing multiple windows support

Supporting multiple windows is doable, but depending on your real-time sync requirements can be quite problematic. The main issue is that if your Electron app is a web-app first, it is extremely likely that the entire data flow is contained within the web application; so by allowing extra windows, you need to solve the problem of windows potentially being out of sync.

The best sync primitive is [Broadcast Channel API](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API) -- it allows to send messages to all other open windows in the same domain (if your windows have apps on different domains, then you either don't need to sync them, or it has to happen via the main process). Here how it can look like:

{% highlight js linenos=table %}
const clientId = generateUUID()
const multiWindowBC = new BroadcastChannel('multiwindow')

multiWindowBC.postMessage({
    type: 'MultiWindowShareData',
    clientId,
    dataVersion: 1,
    data: ...
})

multiWindowBC.addEventListener('message', (event) => {
    if (!event.data || event.clientId === clientId) return
    if ('dataVersion' in event.data) processBroadcastedData(event.data)
})
{% endhighlight %}

Key things to look out for:

- every window, including the one which sent the BroadcastChannel message, receives it, so you need to filter them out
- when sending the message, it is serialized using [structuredClone](https://developer.mozilla.org/en-US/docs/Web/API/Window/structuredClone), so if you pass any custom classes, they will be converted using implemented `toString()` methods
- if you save your data to localStorage/IndexedDB, I recommend to do so only from a single window. Similarly, I would not recommend to read updated data from them, as that will require reading and deserializing the data. What is even worse, you are likely relying on referential equality for optimized re-rendering, and that would thrash the current data collections
- if you have an active session in your app, you probably want to close all non-active windows when the user logs out -- this moment is particularly vulnerable to race conditions if multiple windows can write to the storage

I'd advise against relying on eventual consistency via HTTP/web socket requests. While it works pretty well with good internet, it still takes some time, and while offline or during server load it can cause frustration for power users who will use this feature.

## Resources consideration

Electron uses Chromium process separation model, which is quite hungry for resources, especially RAM; each additional window can easily add 150-250MB of consumed memory. Be especially careful with memory leaks if you use something like [browserWindow.hide()](https://www.electronjs.org/docs/latest/api/browser-window#winhide), as it will remove the window from the list of all opened apps and you won't be able to reach it using `cmd + \`` on macOS. There is a static method [BrowserWindow.getAllWindows()](https://www.electronjs.org/docs/latest/api/browser-window#browserwindowgetallwindows), you can use it to make sure there are no dangling windows in your application.