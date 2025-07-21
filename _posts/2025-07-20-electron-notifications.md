---
layout: post
title: Electron Notifications
keywords: Electron.js, Electron, notifications, notification options, notifications toastXML, notification actions
excerpt: "We can add actions to native notifications from Electron."
---

You can use both Web notifications, the ones you can use in a regular web application executed in a browser context; and you can use native OS notifications. The first option is good because you can reuse the logic between your web and native application, but overall native notifications provide:

- more native look and feel
- allow better customization
- allow more options

> Note: be careful and store references to fired notifications (see [this article](/2025/02/22/electron-mac-notifications)); otherwise they will be garbage collected after some time and interacting with them won't trigger your callbacks.

## macOS notifications

When you create a notification on macOS, in addition to standard `title` and `body`, you can provide an array of `actions` ([docs](https://www.electronjs.org/docs/latest/api/notification#notificationactions)) and add an event handler: `notification.on('action')`. This handler will receive _index_ of the action from the actions array as the second argument.

It is pretty straightforward, here is a full example registering 3 buttons:

{% highlight js linenos=table %}
const notificationOptions = {
    title: data.title,
    body: data.body,
}
notificationOptions.actions = [
    { type: 'button', text: 'Complete a task' },
    { type: 'button', text: 'Snooze for 30 minutes' },
    { type: 'button', text: 'Snooze for 5 hours' },
]
const notification = new Notification(notificationOptions)

notification.on('action', (_, idx) => {
    const activeBrowserWindow = getActiveBrowserWindow()
    switch (idx) {
        case 0:
            completeTask(task)
            break
        case 1: {
            snoozeTask(task, 30)
            break
        }
        case 2: {
            snoozeTask(task, 30)
            break
        }
    }
})

notification.on('click', onClick)

notification.show()
{% endhighlight %}

If the user clicks on the notification itself, `onClick` will be called. If the user selects an action, `notification.on('action', cb)` will be called. One thing to keep in mind is that if you have only 1 action, it will be inlined, otherwise it will be a dropdown menu.

## Windows notifications

> If your application is not packaged (has identity), you'd need to call `app.setAppUserModelId(APP_ID)`, which must match `appId` value in your `electron-builder` configuration (or in your tool for building/packaging the app)

On Windows, the API to provide custom actions is slightly different. Instead of providing custom notification actions and a separate event handler, we utilize custom protocol activation (see [my article](/2025/07/20/electron-apps-custom-protocols)). Windows also allows us to use some system activated actions as well, like `snooze`. You can check out the [full documentation](https://learn.microsoft.com/en-us/windows/apps/design/shell/tiles-and-notifications/adaptive-interactive-toasts?tabs=xml), everything declared in XML should be fully applicable to your application as well. Here is the full list of templates: [ref](https://learn.microsoft.com/en-us/uwp/api/windows.ui.notifications.toasttemplatetype?view=winrt-26100).

> Below I provide a generic snooze option, which should snooze the notification for 10 minutes. You can provide a [select input](https://learn.microsoft.com/en-us/windows/apps/design/shell/tiles-and-notifications/adaptive-interactive-toasts?tabs=xml#snoozedismiss) for the user to customize the value

{% highlight js linenos=table %}
const notificationOptions = {
    title: data.title,
    body: data.body,
}
notificationOptions.actions = [
    { type: 'button', text: 'Complete a task' },
    { type: 'button', text: 'Snooze for 30 minutes' },
    { type: 'button', text: 'Snooze for 5 hours' },
]
notificationOptions.toastXml = `
<toast
    activationType="protocol"
    launch="myapp://notificationTask?id=${data.id}">
    <visual>
        <binding template="ToastText02">
        <text id="1">${data.title}</text>
        <text id="2">${data.body}</text>
        </binding>
    </visual>
    <actions>
        <action activationType="system" arguments="snooze" content="" />
        <action content="Complete Task" activationType="protocol" arguments="myapp://completeItem?id=${data.id}" />
    </actions>
</toast>`

const notification = new Notification(notificationOptions)
notification.show()
{% endhighlight %}

We do not even assign `notification.on('click', cb)` event handler, because the entire notification is described via XML and any click will be handled by our custom protocol, or by the system itself for snoozing.

## Windows Notification Activation Quirk

You might also experience the issue where the application is activated (the app icon in the task bar will be pinged), but the actual app window will not be brought forward. There is an old [Electron issue](https://github.com/electron/electron/issues/2867), which has a few workarounds. I personally use this approach (but I highly recommend to read the issue and see what works in your case):

{% highlight js linenos=table %}
browserWindow.setAlwaysOnTop(true)
browserWindow.setAlwaysOnTop(false)
{% endhighlight %}