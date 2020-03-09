---
title: "Setting up C++"
description: 
date: '2020-03-13'
---

## Setup on Windows

On Windows we can use the [MinGW](https://nuwen.net/mingw.html) C++
distribution. This is a free C++ compiler produced by the GNU
organisation (GW = GNU Windows, Min = minimal).

No editor is included; you can use Notepad++.

The distribution includes some widely used third-party libraries (e.g. Boost, PCRE).

Double-click `open_distro_window.bat` to start a command shell.
Working on the USB stick will be slower than a hard drive, so switch
to the `C:` drive and find a writable directory there.

```cmd
D:MinGW> c:
C:\> cd Users\<TAB>
C:Users\...> 
```

## Setup on MacOS

There are [various
ways](https://www.macobserver.com/analysis/5-ways-to-write-c-code-on-your-mac/)
to install C++ on a Mac.  If you can open a terminal window, try `brew install gcc`.
