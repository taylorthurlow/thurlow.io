---
layout: post
title:  "Using caps lock as CMD and CTRL in macOS, selectively"
date:   2019-01-21 12:00:00 -0800
categories: macos 
---
One of the larger reasons I find myself switching between OSes, and away from macOS in particular, is its inability to play nice with non-standard keyboard layouts. My [HHKB](https://en.wikipedia.org/wiki/Happy_Hacking_Keyboard) has a Control key where you'd expect to find a Caps Lock key (which is great, by the way), so using a standard Mac keyboard on my laptop is jarring. I talked about this in a [previous blog post](/posts/iterm-and-caps-lock-ctrl), where I used iTerm and some shortcut trickery to send ctrl-codes to my terminal whenever it received command key shortcuts. This works but it's pretty messy.

##In Comes Karabiner
[Karabiner](https://pqrs.org/osx/karabiner/) (or more specifically, Karabiner-Elements, is a macOS application which emulates a keyboard device, and allows deep customization. Through Karabiner's "complex modifications" section, we can write a little configuration file which achieves the following goals:

* Change Caps Lock to Command in all applications except your terminal application
* Change Caps Lock to Control only in your terminal application

Once you've installed Karabiner, open it and run it. Now you can open your text editor of choice and create a new file inside `~/.config/karabiner/assets/complex_modifications`. You can name it whatever you like - but add a `.json` extension at the end. Now copy and paste the following JSON into the new file:

~~~~json
{
  "title": "Caps Lock and Kitty",
  "rules": [
    {
      "description": "Caps Lock as Command except in Kitty",
      "manipulators": [
        {
          "type": "basic",
          "from": {
            "key_code": "caps_lock",
            "modifiers": {
              "optional": [
                "any"
              ]
            }
          },
          "to": [
            {
              "key_code": "left_command"
            }
          ],
          "conditions": [
            {
              "type": "frontmost_application_unless",
              "bundle_identifiers": [
                "^net\\.kovidgoyal\\.kitty$"
              ]
            }
          ]
        }
      ]
    },
    {
      "description": "Caps Lock as Control in Kitty",
      "manipulators": [
        {
          "type": "basic",
          "from": {
            "key_code": "caps_lock",
            "modifiers": {
              "optional": [
                "any"
              ]
            }
          },
          "to": [
            {
              "key_code": "left_control"
            }
          ],
          "conditions": [
            {
              "type": "frontmost_application_if",
              "bundle_identifiers": [
                "^net\\.kovidgoyal\\.kitty$"
              ]
            }
          ]
        }
      ]
    }
  ]
}

~~~~

##Tweaking the JSON
There are only a couple things you might need to change for your use case. Firstly, because I'm using a terminal called kitty, you'll see that in the description and title sections. Change those as you like. The other modification necessary is changing the bundle identifier that Karabiner uses to identify which window we are talking about. To do this, head over to your terminal and execute the following command, substituting the application path for the application path of your terminal:

~~~~bash
$> mdls -name kMDItemCFBundleIdentifier -r /Applications/kitty.app
net.kovidgoyal.kitty%
~~~~

The result, in my case, `net.kovidgoyal.kitty`, is the bundle identifier. Notice that I omitted the `%` at the end, which you should as well, if you see one. Now that we know the bundle identifier, we need to change the entries in the JSON I provided earlier. There are two sections in the JSON which read:

~~~~bash
"bundle_identifiers": [
  "^net\\.kovidgoyal\\.kitty$"
]
~~~~

Each entry in the `bundle_identifers` key is a regular expression string. If you know regular expression syntax, then it should be clear why we have the extra symbols in the string - but if you don't, that's fine. The bottom line is that we need to take the bundle identifier we had before, and add a `^` to the beginning, a `$` to the end, and `\\` before every dot.

Once you've changed the identifier values in the JSON, save the file, and restart Karabiner. Head over to the "Complex Modifications" tab, and click "Add rule".

![Imgur](https://i.imgur.com/yROj0r7.png)

Click "Enable All" under the section in the list and the modifications are now active!

Happy hacking!
