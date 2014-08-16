---
title: "Fixed Width Fonts in Gmail"
layout: "post"
permalink: "/2009/03/fixed-width-fonts-in-gmail.html"
uuid: "3722722127690283974"
guid: "tag:blogger.com,1999:blog-32090046.post-3722722127690283974"
date: "2009-03-13 14:09:00"
updated: "2009-03-13 14:16:22"
description: 
blogger:
    siteid: "32090046"
    postid: "3722722127690283974"
    comments: "0"
categories: 
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

I like to compose and read email with a fixed width font. Gmail now supports reading a message with a fixed width font using the *Show in fixed width font* option - but you have to do that on each message as you read it.

If you use Firefox, you can create a user stylesheet that is domain specific to force all non-HTML messages and the message composition textarea to display in fixed width.

The user stylesheet should contain:

```css
@-moz-document domain(mail.google.com) {
    /* GMail messages and textarea should use fixed-width font */
    textarea.dV, div.ii.gt {
        font-family: MonoSpace !important;
        font-size: 9pt !important;
    }
}
```

Place the stylesheet (on MacOS) under your Firefox profile directory e.g. `~/Library/Application Support/Firefox/Profiles/XXXXXXXX.default/chrome/userContent.css`
