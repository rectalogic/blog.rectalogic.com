---
title: "Monoprice 27\" IPS LED Monitor DisplayPort Issue"
layout: "post"
permalink: "/2013/05/monoprice-27-ips-led-monitor.html"
uuid: "7280169808638818654"
guid: "tag:blogger.com,1999:blog-32090046.post-7280169808638818654"
date: "2013-05-24 12:56:00"
updated: "2013-05-24 12:56:38"
description: 
blogger:
    siteid: "32090046"
    postid: "7280169808638818654"
    comments: "3"
categories: 
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

I recently bought some [Monoprice 27" IPS LED](http://www.monoprice.com/products/product.asp?p_id=10489) monitors at work for use with 2013 Retina MacBook Pros. We also bought Monoprice mini-DisplayPort to DisplayPort [cables](http://www.monoprice.com/products/product.asp?p_id=6006).

It turns out there is a problem in MacOS 10.8.3 using this monitor with the DisplayPort connection - the monitor outputs YPbPr mode instead of RGB and the colors all look washed out and wrong. Looking in the System Information app you can see that the display is being detected as a TV (shows "Television: Yes").

I then tried connecting using the provided dual link DVI cable, but using Apple's "Mini DisplayPort to DVI Adaptor" - the colors look correct and "Television: Yes" is not there so this corrects the issue - but of course resolution is limited to 1920x1080 because the adapter is not dual-link. I expect an mDP-dual-link DVI adapter would work fine but those are expensive.

This problem of using YPbPr and detecting the display as a TV when using DisplayPort seems somewhat common and happens with other displays too, I found some threads discussing the issue with other monitors - [Dell U2410f](http://forums.macrumors.com/showpost.php?p=14298758&postcount=165) and [Dell U2713HM](http://forums.macrumors.com/showthread.php?t=1481582).

The fix is to override the [EDID](http://en.wikipedia.org/wiki/Extended_display_identification_data#EDID_1.3_data_format) returned by the monitor and patch byte 24 so the monitor only reports it supports RGB 4:4:4. This forum post has a [patch-edid.rb](http://embdev.net/topic/284710#3027030) script which generates an override file for the display that does that.

This is the process to fix the monitor:

1. Close the MacBook lid so the monoprice display is the only attached display.
1. Run "ruby patch-edid.rb" using the script downloaded from the post in the thread above to generate an EDID override directory for the display - this will override the EDID to force RGB instead of YPbPr.
1. Move the created directory to /System/Library/Displays/Overrides e.g.

    ```bash
    sudo mv DisplayVendorID-6670 /System/Library/Displays/Overrides
    ```

1. Reset ownership, e.g.

    ```bash
    sudo chown -R root:wheel /System/Library/Displays/Overrides/DisplayVendorID-6670
    ```

1. Reboot or power cycle the display - check Display Preferences and ensure that the Display Profile is set to `Display with forced RGB mode (EDID override)`
