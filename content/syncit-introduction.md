+++
title = "Syncit: a privacy-first co-browsing tool"
date = 2020-06-21
category = "syncit"

+++

Today I'm happy to share my new open-source project: [Syncit](https://syncit.luckid.io/), which is a co-browsing tool. It works like TeamViewer in the browser but provides better performance and privacy protection without downloading client.

<!-- more -->

## Why Syncit?

I use remote desktop software, like TeamViewer, frequently in my daily work. There is no doubt that these tools are powerful since they can view and control a remote computer, but for me, they are not perfect.

For example, I always see people quit every instant messaging software before sharing their screen, to avoid the notifications about some personal message.

Also, they would not love me to see their desktop which uses a family photo as the wallpaper and has many sensitive files on it.

And of course, they will not leave the screen during the sharing, to make sure I'm only doing 'right' things.

I think this is all about privacy, which could be hard to solve by the current remote desktop software.

## How Syncit works?

Syncit only works in the browser, which means it can share the view of a website and can do remote control in it, but could not access anything out of the browser.

This is the foundation of Syncit's privacy strategy. Since the browser is one of the most advanced sandboxes, it can guarantee that users will not escape the scope we defined.

And Syncit's way to share the view of a website is quite different from the graphic-based approach.

It uses [rrweb](https://www.rrweb.io/) under the hood to take a snapshot of the DOM, and keep recording mutations as an op-log. This implementation has been proved that it can provide pixel-perfect, low-latency, low-bandwidth screen sharing.

You can refer to Syncit's [design doc](https://github.com/Yuyz0112/syncit/blob/master/docs/design.md) for further details.

## What's next for Syncit?

I do not think Syncit will replace the traditional remote desktop software, but it could be quite useful in some scenarios.

Currently, it is at an early development stage, and I would love to hear from the community for more feature requests.
