---
layout: post
title:  "Replicating macOS's screenshot utility on Linux"
date:   2017-11-10 12:00:00 -0800
categories: linux 
---
I am *really* picky about my operating systems, and often find myself in a situation where I'm not satisfied with using Linux, macOS, or Windows. This is mostly my own fault, and the subject for a different blog post entirely. Development on my laptop (a 2015 Macbook Air running macOS Sierra) and development on my home desktop (Ubuntu 16.04 with [i3wm](https://i3wm.org/)) are two different beasts. Generally, I prefer the setup I have on my desktop, but screenshots are one front where macOS [blows Linux out of the water](https://support.apple.com/en-us/HT201361). A user on AskUbuntu [posted](https://askubuntu.com/a/352276/540640) about just this, I've made some small modifications and would like to share.

## Setting Up

We'll need a few things for the script to work properly:

* `scrot`
* `imagemagick` (for `convert`)
* `libnotify-bin` (for `notify-send`, not needed if you don't want the notification)

On Ubuntu, do a simple `apt-get install scrot imagemagick libnotify-bin`. Substitute for your applicable package manager if you use a different flavor of Linux.

With those installed, grab [the shell script here](https://gist.github.com/taylorthurlow/3b544b2a10708124edff969bfbc6c913). It would be wise to place it somewhere in your `PATH` so you can call it in a terminal without providing an absolute path. Here's the contents, which you might need to tweak a bit:

~~~~bash
# dropshadow.sh

#!/bin/bash
SCREENSHOTFOLDER="$HOME/screenshots"

FILE="${1}"
FILENAME="${FILE##*/}"
FILEBASE="${FILENAME%.*}"

# drop shadow: 60% opacity, 10 sigma, +0x +10y
convert "${FILE}" \( +clone -background black -shadow 60x10+0+10 \) +swap -background transparent -layers merge +repage "$SCREENSHOTFOLDER/${FILEBASE}.png"

notify-send -u low -t 2 "${FILEBASE}.png saved."

rm "$FILE" #remove this line to preserve original image
~~~~

Be sure to make the script executable with `chmod +x /path/to/dropshadow.sh`. Open the script in your preferred text editor.

## Configuration

Firstly, configure `SCREENSHOTFOLDER` to a path which you'd like the screenshots to be saved to. Make sure that the directory already exists.

If you'd like you can configure the drop shadow. There are a few parameters to tweak, and the comment on line 29 might make the syntax slightly clearer:

* The `opacity` value is a percentage integer value from 0 to 100, 0 being totally transparent, and 100 being totally opaque
* The `sigma` value is an integer value, essentially the level of blur. 0 is a "hard shadow", and increasing values make the shadow "softer".
* The last two values represent x and y offsets: positive x moves the shadow rightwards, and positive y moves the shadow downwards.

You can read the [imagemagick documentation](http://www.imagemagick.org/Usage/blur/#shadow) for more information on the drop shadow syntax.

By default, the script uses `notify-send` to send a notification to your window manager, with a low priority and a 2 second expiration. I did this because often the `convert` process takes a second or two. Remove or comment this line if you'd like to disable this.

## Running the Script

Now that the script is configured, run it by doing:

~~~~bash
scrot -szb -e 'dropshadow.sh $f'
~~~~

If the script isn't in your `PATH` like I mentioned earlier, you'll need to add it's path to the argument, for example:

~~~~bash
scrot -szb -e '/home/myuser/scripts/dropshadow.sh $f'
~~~~

Either single-click on a window you'd like to save, or click and drag a region to save. `scrot` handles both cases. Once you've got that working, you'll probably want to add a keyboard shortcut. I'm using [i3wm](https://i3wm.org/), but you might not - so this process will be different depending on your window manager. If you're using GNOME, you can [try this](https://askubuntu.com/a/525495/540640). Otherwise, just hit up Google and I'm sure there will be a million answers for your use case.

In mine, I edit `~/.config/i3/config` and add a single line:

~~~~bash
bindsym --release $mod+Shift+s exec scrot -szb -e 'dropshadow.sh $f'
~~~~

And reload the configuration. Screenshot away!

![screenshot with shadow](https://i.imgur.com/Dbu3fYT.png)
