---
title: "Installing Apple's Java in Mavericks"
layout: "post"
permalink: "/2014/01/installing-apples-java-in-mavericks.html"
uuid: "2697416428267019828"
guid: "tag:blogger.com,1999:blog-32090046.post-2697416428267019828"
date: "2014-01-24 20:30:00"
updated: "2014-01-24 20:30:23"
thumbnail: "{{ site.baseurl }}/images/java-mavericks/thumb.png"
description: 
blogger:
    siteid: "32090046"
    postid: "2697416428267019828"
    comments: "0"
categories: 
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

Mavericks doesn't have Java installed by default. If you try to use the `/usr/bin/java` executable from Terminal, it will pop up a dialog prompting you to visit Oracle's site and download a JDK.

```bash
$ /usr/bin/java
```

![java-oracle.png]({{ site.baseurl }}/images/java-mavericks/java-oracle-s1600.png)

If you want to install Apple's Java instead, then run java from a subshell. This will pop up a dialog prompting you to directly install Apple's JDK.

```bash
$ `/usr/bin/java`
```

![java-apple.png]({{ site.baseurl }}/images/java-mavericks/java-apple-s1600.png)

The Java installer stub `/System/Library/Java/Support/CoreDeploy.bundle/Contents/Download\ Java\ Components.app` appears to check if it was invoked from an interactive TTY or not, and changes behavior accordingly.