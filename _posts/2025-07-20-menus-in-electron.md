---
layout: post
title: Menus in Electron apps -- Application, Tray, Context
keywords: Electron.js, Electron, application menu, tray menu, context menu, right-click menu
excerpt: "There are multiple menus available in native applications and in Electron, with small tricks."
---

There are 3 types of menus in a typical native application:

- Application menu
- Context menus (right click)
- Tray menu

Let's discuss each one of them. In Electron, all menus are created using the same API (usually [Menu.buildFromTemplate()](https://www.electronjs.org/docs/latest/api/menu#menubuildfromtemplatetemplate)), so we'll discuss the implementation at the end of the article.

## Application Menu 

Titlebars went out of style these days on pretty much every platform, and because of that, at least on Windows, the application menu is not present anymore and settings from it moved to other parts of the application. It is pretty common to be absent on Linux as well.

However, on macOS the menubar (top bar) is still an integral part of the design, so it is important to add every helpful function there and properly localize it. Another important part is that even if we don't show the menu and just register it, all the specified shortcuts (named "accelerator" in Electron's API) will still work.

The API is very simple:

{% highlight js linenos=table %}
Menu.setApplicationMenu(menu)
{% endhighlight %}

Technically, each window needs its own application menu. For example, on macOS it is common to list all open windows and put a checkmark against the currently focused window, and in your menus you can add/remove menu items based on the window's state, like which view is active. You can technically save references to specific menu items and manipulate them in the imperative way, but I personally would recommend to simply re-create the menu unless it is expensive and you do it a lot. But you can totally start with something like this:

{% highlight js linenos=table %}
const browserWindow = new BrowserWindow()

browserWindow.on('focus', () => {
    Menu.setApplicationMenu(createAppMenu(browserWindow))
})
{% endhighlight %}

## Context Menu (right-click menu)

This is a really confusing part of Electron -- by default, it _doesn't_ handle right clicks. So by default, simply nothing will happen when you right click on empty space.

In order to handle the event, we need to register a handler: `browserWindow.webContents.on('context-menu')`. This will give us all the info when the user invoked the context menu: stuff like coordinates, whether it was invoked on a link, whether there is any text selected, what is the media type (image, video, audio, etc) and so on. See the full list of properties [here](https://www.electronjs.org/docs/latest/api/web-contents#event-context-menu).

You can use something like [electron-context-menu](https://github.com/sindresorhus/electron-context-menu) library, or feel free to look into the [source code](https://github.com/sindresorhus/electron-context-menu/blob/main/index.js), it is pretty comprehensive.

A simple implementation would look like:

{% highlight js linenos=table %}
const { dialog, clipboard, shell, Menu } = require('electron')

win.webContents.on('context-menu', (_event, params) => {
    const hasText = params.selectionText.length > 0
    const isLink = Boolean(params.linkURL)

    const menuOptions = [
        {
            label: 'open dialog',
            click: () =>
                dialog.showMessageBox(win, {
                    message: 'Hello from context menu!',
                }),
        },
    ]

    if (hasText) {
        menuOptions.push({
            label: 'copy selected text',
            click: () => clipboard.writeText(params.selectionText),
        })
    }

    if (isLink) {
        menuOptions.push({
            label: `Open link ${params.linkURL}`,
            click: () => shell.openExternal(params.linkURL),
        })
    }

    const contextMenu = Menu.buildFromTemplate(menuOptions)
    contextMenu.popup({
        window: win,
        frame: params.frame,
    })
})
{% endhighlight %}

## Tray Menu

Last, we have tray menus. It is an optional part of the application, and in general it is a good idea to provide it as an option -- some users are highly selective on what should display there (on smaller screens like laptops they can take a lot of space), but also it is often a good idea to provide some general app views, relevant app data and maybe some app info/settings.

Interesting enough, you can create multiple tray menus, but usually it is not the desired behaviour and you want to keep only 1 tray menu. Similar to the application menu, the data can change, and you might want to re-render it fully instead of modifying affected elements one-by-one, as that can be error-prone.

From my experience, both on Windows and macOS the menus are static (meaning that you need to reopen them to apply changes), and on Ubuntu/GNOME the changes are applied immediately, so keep that in mind if you have some rapidly changing data.

The API is pretty straightforward:

{% highlight js linenos=table %}
const tray = new Tray(trayIcon)
tray.setContextMenu(Menu.buildFromTemplate(renderContextMenu()))

// after you are done
tray.destroy()
{% endhighlight %}

## Implementing Menus

Good news is that Electron provides _a lot_ out of the box. While creating all of this sounds might sound pretty daunting, you can start with a limited amount of options, and also you have a lot of options for free. Consider this menu:

{% highlight ts linenos=table %}
const menu: Electron.MenuItemConstructorOptions = {
    role: 'editMenu',
    submenu: [
        { role: 'cut'  },
        { role: 'copy'  },
        { type: 'separator' }
        { role: 'paste' },
        { role: 'pasteAndMatchStyle' },
        { type: 'separator' }
        { role: 'selectAll' },
    ]
}
{% endhighlight %}

This entire menu will be populated with correct titles and it will register platform-specific shortcuts (accelerators in Electron terms), like `cmd + C` on macOS and `ctrl + C` on Windows/Linux. You can see the list of all roles provided by Electron: [ref](https://www.electronjs.org/docs/latest/api/menu-item#menuitemrole). You can also find [helpful types](https://www.electronjs.org/docs/latest/api/menu-item#menuitemtype) like `separator`, `checkbox`, `radio` and `submenu`.

We can provide our own label, our own shortcuts, and our own handlers:

{% highlight ts linenos=table %}
{
    label: 'Open new window',
    accelerator: 'CmdOrCtrl+Shift+N',
    click: (_item, _window, event) => {
        openInNewWindowHandler(event)
    },
    enabled: canOpenNewWindow(),
}
{% endhighlight %}

You can see all available options at the docs: [MenuItem](https://www.electronjs.org/docs/latest/api/menu-item).