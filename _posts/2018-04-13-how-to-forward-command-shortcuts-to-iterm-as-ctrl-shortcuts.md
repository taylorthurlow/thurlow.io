---
layout: post
title: How to forward CMD shortcuts to iTerm as CTRL shortcuts
categories: macos
date: 2018-04-13 12:00:00 -0800
---
After trying a couple different kinds of keyboards, I eventually settled on a [Happy Hacking Keyboard](https://en.wikipedia.org/wiki/Happy_Hacking_Keyboard). The keyboard is really a topic for another post - the reason I mention it is the placement of the `ctrl` key. At first glance, it's gone. Where's it gone to? It's replaced the bastard child of keyboard keys, Caps Lock. I had made the switch to caps lock control around a year before I picked up my HHKB, so it was no problem using it, mostly.

## The Problem

I use macOS, both on my laptop and desktop (a [Hackintosh](https://en.wikipedia.org/wiki/Hackintosh)). Using Linux, or even Windows, the `ctrl` key tends to be the only real modifier key you need to worry about, especially in the context of using a terminal. `tmux` only makes things more complicated. On macOS, the `cmd` key tends to be a pain in the ass.

Copy and paste, quitting applications, and all sorts of other things that you'd typically do with the `ctrl` key on a Linux or Windows machine, all use the `cmd` key instead. So, using macOS's built-in keyboard configuration, I mapped my `ctrl` modifier to be `cmd`, and my `cmd` modifier to be `ctrl`. Sweet, now I can reach for the key I'm used to and get `cmd` instead. 90% of the time this is great.

![macOS Keyboard Settings](/assets/uploads/mexwf19.png)

Where it doesn't work out so well, is in [iTerm](https://www.iterm2.com/). Expectedly, terminal commands like `ctrl + c` and `ctrl + l` (which clears the buffer, super useful) don't just work with the `cmd` key - you need to use `ctrl`. This sucks, because muscle memory from using Linux makes me reach for what is now my `cmd` key. Any terminal programs that use `ctrl` shortcuts are affected (like `nano`, for example).

## The Solution

iTerm is pretty configurable, so what we can do is set up certain key combinations to forward hexadecimal key codes for the 'proper' keys. Open up iTerm's preferences window, and head to the 'Keys' tab. In the 'Key Mapping' section, click the plus button to add a mapping. Click the field to set the keyboard shortcut - let's say `cmd + c`. This is the literal shortcut you'll use to trigger the mapping, so you should be pressing the shortcut that you actually want to be pressing on a regular basis.

![iTerm Key Mappings](/assets/uploads/mz9gb8u.png)

Select the 'Send Hex Code' option in the select box. I use the [Key Codes](https://manytricks.com/keycodes/) app to figure out what the appropriate hexadecimal code is for the key combination that I want to emulate. In this case, we want to emulate `ctrl + c`, so open up Key Codes and with the window focused, press the key combination.

![Key Codes](/assets/uploads/ivmksb3.png)

Copy the hexadecimal code under the Unicode section into the iTerm field. Make sure you include the `0x` prefix - in my case, `ctrl + c` is Unicode `0x3`. Press OK. Now you should be able to test out the new mapping - open up a terminal and use the `cmd + c` shortcut - it should send a SIGINT like `ctrl + c` typically would. Perfect! Now you just need to repeat the process for whatever other `cmd` shortcuts you'd like to map as `ctrl` shortcuts. Here are the mappings I use:

* `cmd + a` `0x1`: My `tmux` command prefix (default is `ctrl + b`, really?)
* `cmd + c` `0x3`: Send SIGTERM
* `cmd + l` `0xC`: Clear terminal buffer (way better than typing out `clear` every time)
* `cmd + o` `0xF`: Nano save
* `cmd + r` `0x12`: Command reverse history search
* `cmd + w` `0x17`: Nano where/search
* `cmd + x` `0x18`: Does a few things, check out [this SO answer](https://stackoverflow.com/a/14255401/1177200).
* `cmd + z` `0x1a`: Suspend process

Obviously you can pick and choose which to add, or add more if you have a need. Also keep in mind that these shortcuts can no longer be used to manipulate iTerm itself, so be sure to avoid mapping any keyboard shortcuts that you already tend to use, like `cmd + q` to quit, or `cmd + t` to open a new terminal tab.

Happy hacking!