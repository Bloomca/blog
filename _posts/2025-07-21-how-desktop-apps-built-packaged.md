---
layout: post
title: How Desktop apps are built and packaged
keywords: Electron.js, Electron, native apps, building, packaging
excerpt: "How a typical Desktop application is built and packaged?"
---

Before we can look at a typical building/packaging process of an Electron application, let's take a more general overview at what does it mean to build and package any Desktop application.

## What is a Desktop application?

At the core, a Desktop application is just a simple binary, similar to any CLI application you use from a terminal. It uses separate APIs to mark it as a GUI app, to create and manage windows, to subscribe to the messages like keyboard and mouse events, and to use graphics APIs, but at the end of the day, it is still normal system libraries like Win32 and Cocoa.

However, users expect to be able to use GUI applications from their desktop environment, not just the terminal. This means that an application needs to package an icon, somehow add itself to the startup list (so it is available in the Launchpad or Start menu), register itself to handle [custom protocols](/2025/07/20/electron-apps-custom-protocols), store a few more icons for the tray menu, potentially a few images and so on.

All of this naturally leads to creating some sort of installer; as an additional benefit, it will put our application to some known folder.

## Security considerations

Historically, GUI apps were ran as a current user, having exactly the same privileges as you. This means that a typical application could just read the entire filesystem, modify almost everything and you wouldn't even know.

Over time we received things like elevated permissions when the app tried to access something it probably shouldn't, macOS rolled out TCC (Transparency Consent and Control) framework which prompts the user when the app tries to access things like camera, mic or system audio, or accesses an important library like Documents.

Arguably, a typical application still has too broad access to the user's resources, so there are additional features like package formats with manifests where the app has to declare all the capabilities (and the system can still ask the user before granting access), and almost all storefronts additionally sandbox the applications.

## Codesigning

Another layer of security is knowing who is behind the application you just downloaded. In a way, it is a similar procedure to a typical web application where we provide certificates for domains, but in this scenario, the central authority will the be the platform holder (Apple/Microsoft). Due to this reason, codesigning is not a thing for Linux packages (although storefronts like Snap and Flathub remove that necessity due to review process).

Having a single authority allows platform holders to revoke the certificate, which would show the warning and prevent installing the application in case it proves to contain malware.

## Electron building and packaging process

Alright, now that we know the historical trajectory of the apps, where does it leave the Electron apps? The reality of modern Desktop app distribution is that you need to target multiple storefronts and architectures, as people have very different opinions and expectations how their software should be handled by the OS.

Fundamentally, it is not _hard_ to build the Electron application -- in fact, it is already pre-built for us (that's why Electron itself is a devDependency). All we need to do is transpile our code and put it together with the assets into an Electron archive. Things get a bit more complicated if we want to include some native code, but since the main binary is pre-compiled, our main option (unless you want to recompile Electron) is to include the compiled code using Node addon.

After we've built the application, we need to package it for each distribution method. Tools like [electron-builder](https://www.electron.build/index.html) and [electron-forge](https://www.electronforge.io/) handle most of this complexity, but understanding what they're doing helps when things go wrong.

## Windows

Windows puts a lot of focus on backwards compatibility, and because of that it is still extremely common to distribute the applications using the old way, where we basically get an application installer which is run as a regular application. That installer will create all the folders, copy all the files and update the registry values. This type of distribution is called Win32 apps, and in `electron-builder` it is supported with [NSIS](https://en.wikipedia.org/wiki/Nullsoft#Nullsoft_Scriptable_Install_System) installer.

> Fun note: it was developed by the company who made WinAMP!

However, Microsoft acknowledges that this method has its own issues, and generally recommends to package applications (which gives them identity) for a set of [features](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/modernize-packaged-apps). However, even for publishing to the MS Store, you can still use Win32 application distribution method.

## macOS

There are 2 main ways to distribute your GUI application on macOS:

- publish it to the Mac App Store
- distribute it on your in the DMG (disk image) format

While there are differences on how the apps behave depending on the distribution method, the packaging process is pretty similar. All macOS apps need to bundled, contain an `Info.plist` with all capabilities specified, and need to be codesigned using a correct certificate (which you need to generate directly from Apple using a dev account).

After that is done, you can immediately distribute your DMG version, and you can submit your MAS build. MAS applications _always_ run in the sandboxed environment, where they only access "fake" filesystem, some operations like custom AppleScript targets is not allowed (you need to declare all apps you want to interact with using AppleScript) and so on.

## Linux

On Linux, there is no central authority, and because of that there are multiple unrelated efforts to ease the distribution efforts (I'll specify a few major ones, but there are more):

- [FlatPak](https://flatpak.org/) -- provides a custom shared runtime layer, which nearly guarantees that your app will run on almost any distro. Applications run in a sandbox and need to declare all capabilities they require
- [Snapcraft](https://snapcraft.io/) -- similar idea, there is a base layer which simplifies dependency management. Strongly tied to Canonical (company behind Ubuntu), also runs apps in a sandbox and requires capabilities declaration. If your application needs some extra capabilities, like microphone access, you need to request automatic connection following [this procedure](https://snapcraft.io/docs/process-for-aliases-auto-connections-and-tracks)
- [AppImage](https://appimage.org/) -- a single file format, which is probably closest to portable Win32 applications. Requires 3rd party software to integrate the icon and custom protocols (the `.desktop` file is contained inside the AppImage file)

In general, it is common to provide at least one of these options. Additionally, many Linux users prefer native package formats (`.deb` for Debian/Ubuntu, `.rpm` for Fedora), but these do not have cross-distro compatibility, which each format I mentioned does.
