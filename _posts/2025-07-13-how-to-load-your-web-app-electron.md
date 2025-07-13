---
layout: post
title: How to load your web application in Electron
keywords: Electron.js, Electron, loading web app, updating web app, handle Electron errors
excerpt: "You can load web application in Electron using local files or pointing to a domain"
---

As I mentioned in the [Electron overview article](/2025/07/13/electron-overview), one of the key advantages of Electron is that you can wrap your existing web application, and likely, it will just work™. You have 2 main approaches on how to load the application:

1. Load it from local files ([browserWindow.loadFile](https://www.electronjs.org/docs/latest/api/browser-window#winloadfilefilepath-options))
2. Load it from your domain ([browserWindow.loadURL](https://www.electronjs.org/docs/latest/api/browser-window#winloadurlurl-options))

In general, Desktop applications are expected to show something even if you are offline, and some have partial functionality. It is also possible to have nearly full functionality, but if you are expanding your web application, it is very likely not the case here.

## Which approach is better?

Fundamentally, loading from local files is a better approach:

- You can design the offline screen using your app’s components and style
- Faster startup time
- Reading from filesystem is more reliable than the network

But it also has a few disadvantages:

- You might need to change the CORS policy on your server
- You might need to change how you build your web application
- Your users will only receive updates when you release a new Desktop application

Let’s talk about the last point in more detail.

## Updating the web app

Another key advantage of Electron is that if you load your files from a domain, your users will get the latest UI fixes and features “for free”. This is very different from a typical desktop application, where all updates are bundled into periodic app updates, also similar to iOS/Android apps (although some formats support lightweight delta updates, it still requires a separate process to rewrite/update source files, sometimes asking for the user input).

As I mentioned in the comparison before, loading local files is a more native approach, but in that case you either lose this benefit, or you need to write your own logic to replace the web app files in the background.

In some cases it makes sense to go with the files approach, but if you have consistent web app updates and more sparse desktop app updates, it is a very big benefit to have automatic parity between web and desktop features. In that case and if you want to keep Electron as a relatively thin wrapper, it might make sense

## Handling loading failures

In both local files and loading it from the network, it is possible that something will go wrong, although the network is usually far less reliable. You can technically inline your entire HTML + CSS + JS application into a string and guarantee that it works as long as your JS files are loaded correctly, but I assume it is not the best course of action, as it will increase startup time due to Node needing to read and parse a much bigger file.

When the app fails to load the HTML page, nothing visible happens by default. It is your responsibility to indicate to the user that something went wrong, otherwise they will only see a blank window.

You can determine that something went wrong in this callback:

{% highlight js linenos=table %}
browserWindow.loadURL('https://example.com').catch((error) => {
    // handle the error here
})

browserWindow.loadFile(path.join(__dirname, 'index.html')).catch((error) => {
    // handle the error here
})
{% endhighlight %}

The `error` object will contain error code. You can see all options in [did-fail-load](https://www.electronjs.org/docs/latest/api/web-contents#event-did-fail-load) event (although the format is slightly different there, the values are the same). The tricky part is that not all error codes are equal, and you can see the full list in the Chromium source code: [ref](https://source.chromium.org/chromium/chromium/src/+/main:net/base/net_error_list.h).

Here is an excerpt:

{% highlight c linenos=table %}
// Ranges:
//     0- 99 System related errors
//   100-199 Connection related errors
//   200-299 Certificate errors
//   300-399 HTTP errors
//   400-499 Cache errors
//   500-599 ?
//   600-699 <Obsolete: FTP errors>
//   700-799 Certificate manager errors
//   800-899 DNS resolver errors
//   900-999 Blob errors

// An asynchronous IO operation is not yet complete.  This usually does not
// indicate a fatal error.  Typically this error will be generated as a
// notification to wait for some external notification that the IO operation
// finally completed.
NET_ERROR(IO_PENDING, -1)

// A generic failure occurred.
NET_ERROR(FAILED, -2)

// An operation was aborted (due to user action).
NET_ERROR(ABORTED, -3)

// An argument to the function is incorrect.
NET_ERROR(INVALID_ARGUMENT, -4)

// The handle or file descriptor is invalid.
NET_ERROR(INVALID_HANDLE, -5)

// The file or directory cannot be found.
NET_ERROR(FILE_NOT_FOUND, -6)

// An operation timed out.
NET_ERROR(TIMED_OUT, -7)
{% endhighlight %}

The tricky part is that some actions don't need to be handled. For example, if the error code is `-3`, it can be due to the user closing the window, pressing `cmd + R` to reload the page, or simply because the app redirected to another page before all assets loaded -- in that case you shouldn't do anything! In case like `-5` or `-6` (invalid file descriptor or not found file) you might want to recommend to reinstall the application, and in most cases you'd just want to build a simple page which will show the error to the user, so they can either change something in their setup or give you more information.

You have several ways to guarantee that it will display correctly for the user:

- create the page as a separate file (might be too much overhead to pass the error code and description)
- inline the page into a string and load as a string
- use Electron's [dialog API](https://www.electronjs.org/docs/latest/api/dialog), which uses native system dialog boxes
- send a notification (please keep in mind that the user might not receive it for multiple reasons, so it is the least reliable option)

The best option is probably to show a dialog.