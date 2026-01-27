---
title: "Switching to Wayland"
date: 2022-07-27T22:09:53+05:30
draft: false
tags: ["Linux", "Ricing", "Wayland"]

cover:
    image: "cover.png"
    alt: "Description of image"
    relative: true
---

---

Hello! 11 days ago I switched from x11 to wayland, because 60 FPS scrolling is a feature too good to miss.

Now what _is_ wayland, you may ask? Wayland is like a replacement to the traditional x11 display protocol. You can read more about it [here](https://wayland.freedesktop.org/).

My x11 setup consisted of the following software:

-   [DWM](https://github.com/rv178/dwm) as the window manager.
-   [Polybar](https://github.com/polybar/polybar) as the status bar.
-   [Dunst](https://github.com/dunst-project/dunst) as the notification daemon.
-   [ST](https://github.com/rv178/xelph-st-git) as my terminal emulator.
-   [Betterlockscreen](https://github.com/betterlockscreen/betterlockscreen) as the lockscreen.
-   [Rofi](https://github.com/davatorium/rofi/) as the app launcher.

Now before switching to wayland, some of these tools listed above straight up break in a wayland environment, like betterlockscreen
and rofi. ST also behaves weirdly on wayland, but it works. And of course, since DWM is an x11 window manager,
you cannot use it on wayland.

## Choosing a compositor

[Sway](https://github.com/swaywm/sway) was an awesome choice, but I didn't really like the i3-ish aspect of it.

So I was looking for something more DWM-like.

[River](https://github.com/riverwm/river) was a great replacement.

## Choosing a bar

There are other alternatives like [yambar](https://codeberg.org/dnkl/yambar), but I chose [waybar](https://github.com/Alexays/Waybar).

## Notification daemon

While dunst works in wayland, I opted for a notification daemon called [mako](https://github.com/emersion/mako) instead.

## Terminal emulator

ST was a great option, but I had to part ways with it. I switched to the [foot](https://github.com/DanteAlighierin/foot) terminal instead.

## Lock screen

Since betterlockscreen does not work on wayland, I chose [swaylock](https://github.com/swaywm/swaylock) for the lock screen.

## Application launcher

I use [rofi-lbonn-git](https://github.com/lbonn/rofi) for the app launcher.

## My setup

![Alt](https://raw.githubusercontent.com/rv178/.dotfiles/wayland/.assets/screenshots/2.png)

![Alt](https://raw.githubusercontent.com/rv178/.dotfiles/wayland/.assets/screenshots/1.png)

You can find my dotfiles [here](https://github.com/rv178/.dotfiles).

That's it for this blog see you soon!

---
