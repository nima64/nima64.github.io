---
title: Summer of code with Video Lan
description: 
date: 2024-06-01
bannerImage: '/img/VLC-logo.jpg'
tags:
  - GSOC
  - C 
  - Lua
---

## Overview of the Code

[Updated SmartLoad extension to support VLC 4.0](https://github.com/thebamby/vlc-SmartLoad/pull/1)  
[Auto-run branch](https://code.videolan.org/nima64/vlc/-/tree/autorun-module?ref_type=heads)  
[Test improvements branch](https://code.videolan.org/nima64/vlc/-/tree/tests-improvements?ref_type=heads)  
[Remove traces of CMD_SET_INPUT branch](https://code.videolan.org/nima64/vlc/-/tree/CMD_SET_INPUT-removal?ref_type=heads)  
[Fixed Meta_Changed function from segfaulting branch](https://code.videolan.org/nima64/vlc/-/commit/cb2244b6f81d5c38e9aaddaf83aee09cd0905693)

## Background

### Building VLC

The latest repo is on [gitlab](https://code.videolan.org/videolan/vlc/) they have two build systems **meson** and **gnu autotools**.  

I ended up choosing meson because it generates a ninja builds which is significantly faster on my system than auto tools. Ninja also produces a compile_commands.json which is need for clangd setup.

A blocking point in building vlc was finding the right the packages. Specifically qt packages which sometimes couldn't be recognized by pkg-conf. Eventually I found the packages from Ubuntu 21.10 worked, alternatively I could have built QT from source. I wasn't comfortable building QT, so I stuck with the latter.

I found the correct packages to install from the looking at the QT build decencies in *vlc/modules/gui/qt/meson.build*. I would also look in vlc's docker images for finding linux dependencies.

> Note: to generate a compile_commands.json on autoools use bear.  
> And run `bear -- make`

```js
modules: [
    'Core', 'Gui', 'Widgets', 'Svg', 'Qml', 'QmlModels',
    'QuickLayouts', 'QuickTemplates2', 'QmlWorkerScript',
    'Quick', 'QuickControls2', 'ShaderTools'
    ],
```

### Comprehending the code base

Some useful resources  
[https://wiki.videolan.org/Hacker_Guide/]()   
[https://mfkl.gumroad.com/l/libvlc-good-parts]()   
[https://www.doxygen.nl/manual/]()  

I would recommend compiling your own documentation for source graphs. There are libvlc and libvlccore doxygen documentation online, but that didn't include any of the extensions system.

### VLC Extension System

Since my project is based around the lua extension system I will attempt to explain and show how it works in a brief paragraph.  

The key part in the extension system that controls the communication between vlc and extension code is the *extension manager*.
The extension manager holds all the extension data and contains a function, called control, which is the api that will handle commands. The function gets passed different command enums such as activate, deactivate, is_active... etc. And how the commands get handled is dependant your implementation.  

The implementation from the lua module says that commands will get pushed onto queue. And on separate thread "extension thread" will execute commands and trigger a function in the lua code.
{% image "./extension-system-diagram.jpg" ,"test"%}

## What I did

### Removing CMD_SET_INPUT

[my mentors work]( https://code.videolan.org/videolan/vlc/-/merge_requests/3240)

[Remove traces of CMD_SET_INPUT branch](https://code.videolan.org/nima64/vlc/-/tree/CMD_SET_INPUT-removal?ref_type=heads)  

This was a continuation of my mentors work deprecating CMD_SET_INPUT signals through ui commands

### Adding missing Testing

[Test improvements branch](https://code.videolan.org/nima64/vlc/-/compare/master...tests-improvements?from_project_id=435)

One of the goals for my project is to create tests for the extension commands. Only playing_changed had been done the rest needed to be implemented. Testing each command is just checking if the callback in the lua code got executed. The is achieved through blocking our main thread with a semaphore until the lua callback is triggered, from extension thread, signals to unblock the code.  

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

### Auto-run feature

[Auto-run Merge Request](https://code.videolan.org/videolan/vlc/-/merge_requests/5795/commits#note_448656)  

Ideally extensions should be isolated in their own process and have a proper permission system which prompts the user when it uses features that are possible security risks like accessing the filesystem.
Currently the new Autorun Start function is not 100% done, it loads the data and the starting of the new extension threads and enabling marked extensions is naively done through the extension manager.  

Future Work

- AutorunStart should create marked extension threads and enabled them. 

- Extension thread should also be stopped by the last module using the state shared_ptr.

{% image "./autorun-diagram.jpg" ,"test"%}

## Conclusion  

The thing I struggled with the most is being a effective communicator and learning how to communicate your ideas effectively and clearly is very difficult the idea in your head might be different than what someone else's. Being able to show what you mean so both parties can come to consensus is very important to a successful collaboration. For complex module I found sharing diagrams to express the general idea first before proceeding to code was a huge time saver, so my mentor could check the validity of the idea before I started coding.

When I was freelancing couple years ago I worked on a project making a VLC extension for some movie producer. A lot of the documentation had been dated, so I was left to dig up the code base. Back then I didn't know a lick of C and always wondered how VLC worked under the hood. Now I know parts of it :) and I had alot of fun coding this summer.  
Thank you Alexandre Janniaux for your patience with me, teaching me to be better communicator, and being an amazing mentor. And thank you Jean-Baptiste Kempf for making this possible.
