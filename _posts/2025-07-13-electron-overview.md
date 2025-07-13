---
layout: post
title: What is Electron?
keywords: Electron.js, Electron, overview, Chromium
excerpt: "A high-level overview of Electron's components"
---

[Electron](https://www.electronjs.org/) is a cross-platform framework which allows to develop Desktop applications. They are “native” in a sense that you will build and package the installer for your users, but the integration with OS is limited to what the framework provides, and making custom integrations is not trivial, because the application logic is written in JavaScript and executed by Node.js, so if you want to access OS capabilities, you’ll need to write a Node addon. We’ll talk about that later, but it is not the most straightforward thing, although it can definitely be done.

## Limitations

First, let’s talk about limitations. Electron combines two main technologies: Chromium and Node, and when you build and package your application, it includes both. Essentially, you package the entire browser (excluding the tab management, settings, etc) when you ship an Electron application.

Shipping the browser also means that we ship its security model. Back in the day, a single JS crash would kill all browser tabs, but today each tab crashes independently. In Electron, this means that every window is its own process and there is no easy way to share data between each window. There is IPC helper implementation, and if each of your windows is on the same domain, you can use [broadcastChannel API](https://developer.mozilla.org/en-US/docs/Web/API/Broadcast_Channel_API), but that is inter-process communication with its own set of downsides like non-shared memory and no way to pass any non-standard data.

Another thing to keep in mind is that Electron is normally a pre-compiled binary, and that is what the final binary application is unless you compile the source code by yourself. This means that it if you need to handle some OS-level events (e.g. extend [AppDelegate](https://developer.apple.com/documentation/uikit/uiapplicationdelegate) on macOS), you might need to re-compile Electron with your native code statically linked.

## What is a truly native application

Usually people refer to native applications as ones which were built with the OS provided framework, like SwiftUI on macOS and WPF/WinUI on Windows. These frameworks provide rich controls and standard OS interactions out of the box, shortcuts, often some common window behaviour, animations, etc. So all applications built with the OS provided framework will have very similar looking components, will have common behaviour, e.g. think of any native applications the OS is shipped with. This is not as prominent on Linux, but both macOS and Windows feature a lot of uniformity in their apps, especially on macOS.

Luckily, the window system is decoupled from these frameworks, so we can get it in Electron apps as well, but a lot of things will never look and feel like a native app. For example, if we enable multiple windows, `cmd + \`` on macOS will switch between any open window, and on Windows we'll automatically get the snap options if you hover over the maximize button.

These frameworks also have very different rendering engine implementations. While the browser originally aimed at documents and provides very flexible instruments to dynamically adjust your content, native frameworks in general are much more strict as they are aimed to build applications which allows them to achieve perfectly smooth scrolling and nice animations. They also provide some nice default controls like virtualized lists, sticky headers and so on, while in the browser we have to re-implement it or use a library for that.

Last, native applications are compiled to low-level code (not always to actual machine instructions, e.g. C# is often run by .NET Runtime), which allows them to be very performant.

## Advantages

At this point, you might ask: “well, why would I actually use Electron?”. The main reason to use Electron is that despite all the drawbacks, it is a truly cross-platform framework which minimizes differences between platforms. For example, [Tauri](https://v2.tauri.app/) (at least at the time of writing, July 2025) relied on the system WebView (essentially Microsoft Edge on Windows and Safari on macOS) and uses Rust to compile the app code to machine instructions, which allows the binaries to be very compact, like ~10-20MB vs ~50-90MB with Electron.

However, the system WebView can be very different on Windows 8 vs on some Linux distro vs very old macOS version. This is especially bad on Linux (see this [GH Issue](https://github.com/tauri-apps/tauri/issues/3988)), but debugging issues specific to old OS versions is not fun on Windows and macOS as well. Interesting that Tauri is exploring including the frontend runtime with the binaries ([ref](https://v2.tauri.app/blog/tauri-verso-integration/)) to help with this exact issue, but it will definitely increase the size.

Another advantage is that since you can package your app with a very recent Chromium version, you can be very confident that it will work basically the same way it works in a regular Chromium-based browser (Chrome, Brave, Microsoft Edge). Last, sometimes having an application which looks and works exactly the same way across any platform is the desired behaviour, and Electron allows to achieve it with relatively little effort. And by little I do mean it -- you mostly need to adjust the login flow for security reasons, add basic OS integrations which make sense for your application (custom right-click menu, tray menu if makes sense, etc), and it will likely work well. Using any other non-web framework will require you to spend a lot of effort to port and maintain the UI changes.
