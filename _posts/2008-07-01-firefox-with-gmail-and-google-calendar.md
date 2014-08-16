---
title: "Separation of Web and App"
layout: "post"
permalink: "/2008/06/firefox-with-gmail-and-google-calendar.html"
uuid: "4766216136616838981"
guid: "tag:blogger.com,1999:blog-32090046.post-4766216136616838981"
date: "2008-07-01 02:46:00"
updated: "2008-07-01 14:11:49"
description: 
blogger:
    siteid: "32090046"
    postid: "4766216136616838981"
    comments: "0"
categories: [firefox, gmail]
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

I need to run Gmail and Google Calendar in a separate browser instance from my working browser so they are isolated from crashes/hangs. I tried the Firefox 3 [Prism plugin](https://addons.mozilla.org/en-US/firefox/addon/6665) along with [gmail.webapp](http://starkravingfinkle.org/projects/webrunner/gmail.webapp) and [gcalendar.webapp](http://starkravingfinkle.org/projects/webrunner/gcalendar.webapp) but wasn't happy with this - mainly because I don't want a separate instance for gmail and calendar. I want my browser and one other instance hosting both gmail and calendar.

You can run multiple Firefox instances by directly launching `/Applications/Firefox.app/Contents/MacOS/firefox` - but risk corrupting shared profile database files.

So I ended up creating a simple profile just for these webapps - launching the [Profile Manager](http://kb.mozillazine.org/Opening_a_new_instance_of_Firefox_with_another_profile) with
`/Applications/Firefox.app/Contents/MacOS/firefox -P -no-remote` and creating a new profile named google.

Now I can launch gmail and calendar using this new profile in a separate Firefox instance like this: `/Applications/Firefox.app/Contents/MacOS/firefox-bin -P google -no-remote https://mail.google.com/mail https://www.google.com/calendar/render`. I packaged this shell script in a bundle as Google.app and gave it the icon borrowed from `Google Notifier.app`.  You can download it [here](http://www.rectalogic.com/downloads/GoogleFirefox.dmg).

Also, if you prefer monospace fonts when viewing and composing plain text emails in Gmail, create a user CSS file in the google profile you created above (e.g. `~/Library/Application Support/Firefox/Profiles/*.google/chrome/userContent.css`) with the following contents:

```css
@-moz-document domain(mail.google.com)
{
    /* GMail messages and textarea should use fixed-width font */
    .ArwC7c, .iE5Yyc {
        font-family: MonoSpace !important; 
        font-size: 9pt !important; 
    }
}
```
