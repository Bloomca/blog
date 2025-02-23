---
layout: post
title: How to fix Electron notifications not working on macOS after some time
keywords: electron, macOS, notifications
excerpt: "If you don’t save the reference to the fired notifications, the handlers might be garbage collected"
---

Notifications in Electron are provided through a native bridge (although you can use the Chromium web-browser ones, but they are more limited) so they feel more native to the platform. While this is a good thing, they definitely have a few gotchas, and here I will show you one simple pitfall: notifications stop reacting after a minute or two if you don’t activate them immediately. Consider this typical notification code:

{% highlight js linenos=table %}
import { Notification } from 'electron'

function showNotification(title, body, onClick) {
    const notification = new Notification()
    notification.on('click', onClick)
    notification.show()
}
{% endhighlight %}

When you test this code, it will work as expected – the notification will be shown and the `onClick` handler will be called. However, if you let it sit there for a few minutes (either in the Notification Center, or if you set notifications to "alert" type, where they don’t disappear until dismissed manually), you’ll notice that at the very least the active handler is not triggered consistently.

The problem is that the code for that `onClick` handler will be garbage collected if we don’t save the reference to the notification somewhere. So, to keep it safe, we need to implement storage for notifications, and ideally clear it once the notification is handled, something along these lines:

{% highlight js linenos=table %}
import { Notification } from 'electron'

// store a list of notifications so handlers are not garbage collected
let notificationsList = []

function showNotification(title, body, onClick) {
    const notification = new Notification()
    notificationsList.push(notification)
    notification.on('close', () => clearNotification(notification))
    notification.on('click', () => {
        onClick()
        clearNotification(notification)
    })
    notification.show()
}

function clearNotification(notificationToDelete) {
    notificationsList = notificationsList.filter(
        notification => notification !== notificationToDelete
    )
}
{% endhighlight %}

Please note that `close` and `click` events do not guarantee that every dismissing action will trigger one of them. So if you see that it affects the memory consumption, feel free to implement some sort of cleanup after certain period of time.