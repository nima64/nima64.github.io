---
title: First 3 Weeks of summer of code with VLC player
description: 
date: Created
image: '/img/VLC-logo.jpg'
tags:
  - GSOC
  - C 
  - Lua
---
## First Week; Building VLC
With the current repo of vlc player located in their [gitlab](https://code.videolan.org/videolan/vlc/) they have two build systems **meson** and **gnu autotools**.  

The main advantage with meson is the ninja builds it produces which are *super* fast, autotools is much slower but you they allow you to easily disable and enable dependcies the cli. And currently meson build doesn't work for macos.

I ended up to picking meson that it produces a compile_commands.json needed for clangd lang server.

Some big problems I ran into was finding the right packages, alot of the times it I would install the package but it wouldn't register because pkg-conf couldn't find it. So I made sure installed either lib or dev packages.

I found myself in a rut tyring to find the right packages for QT because the script would say I would was missing one of out of the 5 QT packages and I didn't know which one it was. How I ended up find the right packages was by looking at **vlc/modules/gui/qt/meson.build** which contains all the depencies I needed. 
```js
modules: [
    'Core', 'Gui', 'Widgets', 'Svg', 'Qml', 'QmlModels',
    'QuickLayouts', 'QuickTemplates2', 'QmlWorkerScript',
    'Quick', 'QuickControls2', 'ShaderTools'
    ],
```
After compiling my VLC would break everytime I ran my extension which is the whole point of my project. The problem was packages were outaded so my choices were go ubuntu 24.10 or install QT from source. I ended up choosing the former.

## Second Week; Writting Tests
