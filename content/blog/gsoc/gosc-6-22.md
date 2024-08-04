---
title: Summer of code with Video Lan
description: 
date: Created
bannerImage: '/img/VLC-logo.jpg'
tags:
  - GSOC
  - C 
  - Lua
---

## Building VLC

The latest repo is on [gitlab](https://code.videolan.org/videolan/vlc/) they have two build systems **meson** and **gnu autotools**.  

I ended up choosing meson because it generates a ninja builds which is significantly faster on my system than auto tools. Ninja also produces a compile_commands.json which is need for clangd setup.

A blocking point in building vlc was finding the right the packages. Specifically qt packages which sometimes couldn't be recognized by pkg-conf. Eventually I found the packages from Ubuntu 21.10 worked, alternatively I could have built QT from source. I wasn't comfortable building QT, so I stuck with the latter.

I found the correct packages to install from the looking at the QT build decencies in *vlc/modules/gui/qt/meson.build*. I would also look in vlc's docker images for finding linux dependencies.

```js
modules: [
    'Core', 'Gui', 'Widgets', 'Svg', 'Qml', 'QmlModels',
    'QuickLayouts', 'QuickTemplates2', 'QmlWorkerScript',
    'Quick', 'QuickControls2', 'ShaderTools'
    ],
```

## Comprehending the code base

Some useful resources  
[https://wiki.videolan.org/Hacker_Guide/]()   
[https://mfkl.gumroad.com/l/libvlc-good-parts]()   
[https://www.doxygen.nl/manual/]()  

I would recommend compiling your own documentation for source graphs. There are libvlc and libvlccore doxygen documentation online, but that didn't include any of the extensions system.

## VLC Extension System

Since my project is based around the lua extension system I will attempt to explain and show how it works in a brief paragraph.  

The key part in the extension system that controls the communication between vlc and extension code is the extension manager.
The extension manager holds all the extension data and contains a function, called control, which is the api that will handle commands. The function gets passed different command enums such as activate, deactivate, is_active... etc. And how the commands get handled is dependant your implementation.  

The implementation from the lua module says that commands will get pushed onto queue. And on separate thread "extension thread" will execute commands and trigger a function in the lua code.
{% image "./diagram.png" ,"test"%}

## Tests

One of the goals for my project is to create tests for the extension commands. Only playing_changed had been done the rest needed to be implemented. Testing each command is just checking if the callback in the lua code got executed. The is achieved through blocking our main thread with a semaphore until the lua callback is called, from a separate thread, signals to unblock the code.  

```c
typedef enum
{
    CMD_ACTIVATE = 1,
    CMD_DEACTIVATE,
    CMD_TRIGGERMENU,    /* Arg1 = int*, pointing to id to trigger. free */
    CMD_CLICK,          /* Arg1 = extension_widget_t* */
    CMD_CLOSE,
    CMD_SET_INPUT,      /* No arg. Just signal current input changed */
    CMD_UPDATE_META,    /* No arg. Just signal current input item meta changed */
    CMD_PLAYING_CHANGED /* Arg1 = int*, New playing status  */
} command_type_e;

```

## Auto run Module

