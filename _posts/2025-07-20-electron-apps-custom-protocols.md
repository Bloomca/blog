---
layout: post
title: Custom Protocols and Deeplinking in Electron apps
keywords: Electron.js, Electron, custom protocols, protocol handlers, URL schemes, Deeplinking
excerpt: "You can register your app with a custom protocol and call from other places."
---

Each desktop application can register a list of custom protocols it will handle. One application can register multiple handlers, although it is usually not necessary, but can be done for cleaner separation between concerns, like specific views URLs and authentication.

For example, on Windows, you can use [ms-settings:display](ms-settings:display) to open display settings directly. You can either enter it in your browser directly, or type `start ms-settings:display` in the command line.

On macOS, you can use the following link:

- [x-apple.systempreferences:com.apple.preference.universalaccess?Seeing_Display](x-apple.systempreferences:com.apple.preference.universalaccess?Seeing_Display)
- or you can execute it in the terminal with `open x-apple.systempreferences:com.apple.preference.universalaccess?Seeing_Display`

## Why register?

We can register custom protocols for our applications. It can be used in many scenarios:

- Handling login in the OAuth flow
- Opening the desktop application directly from the browser
- Allowing other applications to launch your app directly, including some data
- Modern notifications on Windows can be handled via custom protocols, on macOS we can link widgets to our application, etc

Even if you don't plan on using it a lot, it is a good option to have in order to extend the functionality of your app. There are a lot of applications with custom protocols handlers. For example, on Windows, execute this command in PowerShell:

{% highlight sh linenos=table %}
Get-ChildItem -Path Registry::HKEY_CLASSES_ROOT | Where-Object {
    $_.GetValue("URL Protocol") -ne $null
} | Select-Object PSChildName
{% endhighlight %}

On macOS, you can try the following:

{% highlight sh linenos=table %}
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister -dump | grep -A2 -B2 "scheme:"
{% endhighlight %}

The list won't be exhaustive, but it should give you an idea which apps registered custom protocol handlers.

## How to register custom protocols

You need to do 2 things:

1. call `app.setAsDefaultProtocolClient` with your protocol
2. register custom protocols in your packaging manifest (depends on the platform)

{% highlight js linenos=table %}
if (process.defaultApp) {
    if (process.argv.length >= 2 && process.argv[1]) {
        return app.setAsDefaultProtocolClient(protocol, process.execPath, [
            path.resolve(process.argv[1]),
        ])
    }
    return false
} else {
    return app.setAsDefaultProtocolClient(protocol)
}
{% endhighlight %}

The tricky part here is that only Windows allows to register it purely during runtime (if the app is not packaged). MacOS/Linux require the app to be properly packaged and installed first; if you already have the app installable, it _should_ work as long as you start your dev app with your main app closed.

For macOS, your `Info.plist` should contain this:

{% highlight xml linenos=table %}
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.yourapp.mac.YourApp</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>yourapp</string>
        </array>
    </dict>
</array>
{% endhighlight %}

And on Linux, your `.desktop` file will need to have a line with MimeType and which protocols your app can handle:

{% highlight sh linenos=table %}
MimeType=x-scheme-handler/yourapp;
{% endhighlight %}

## How to handle deeplinks

There is a good tutorial in the [Electron documentation](https://www.electronjs.org/docs/latest/tutorial/launch-app-from-url-in-another-app). The most important part is to understand how different OSes treat additional app launches.

On macOS, the OS enforces a single instance of GUI apps, so when it tries to launch the app when one is already active, it will just activate the currently running instance. To pass that information, we receive the `open-url` event, and it has the calling URL as the second argument:

> If you log your links, please remember to sanitize them

{% highlight js linenos=table %}
app.on('open-url', (event, url) => {
    log.info('Received deeplink:', sanitizeAuthDeeplink(url))
{% endhighlight %}

An important note here is that if your app was not running, you'll still receive this event, after `app.on('ready')` event is fired.

## Windows/Linux

Windows and Linux handle deeplink activation fundamentally different. The default action is to open _another_ instance of your application. If you are wrapping your web application, this is often not what you want, because unlike tabs in a browser, users expect the same desktop app to be synchronized across multiple instances, and it is not trivial to ensure that. You can explicitly prevent it using [app.requestSingleInstanceLock()](https://www.electronjs.org/docs/latest/api/app#apprequestsingleinstancelockadditionaldata):

{% highlight js linenos=table %}
const singleInstanceLock = isMAS() || app.requestSingleInstanceLock()
if (!singleInstanceLock) {
    app.quit()
} else {
    // load your app normally
}
{% endhighlight %}

> Note: Mac App Store (MAS) version can crash on start if you request the lock. See more info in [this Electron issue](https://github.com/electron/electron/issues/15958)

After you requested a single instance lock, you'll receive `second-instance` event and a list of process arguments, and you'll need to parse the list to see if there is a deeplink you can handle:

{% highlight js linenos=table %}
app.on('second-instance', (_, argv) => {
    const deeplinkUrl = argv.find((arg) => arg.startsWith('myapp'))
    log.info('Received deeplink:', sanitizeAuthDeeplink(deeplinkUrl))
{% endhighlight %}

That is not all! In case your app was not running, this event won't be triggered, which makes sense: this is the first instance, after all. To combat that, you basically need to run the same deeplinking function on app's startup and pass `process.argv` directly.

If you don't do so, if somebody obtains a deeplink to your application and click on it (e.g. they use a notification which is kept in the Action Center), it will not respect that and open the default view.