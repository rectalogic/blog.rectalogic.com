---
title: "Installing M2Crypto in a Python virtualenv on Ubuntu 13.10 Saucy"
layout: "post"
permalink: "/2013/11/installing-m2crypto-in-python.html"
uuid: "932149362360686524"
guid: "tag:blogger.com,1999:blog-32090046.post-932149362360686524"
date: "2013-11-01 22:37:00"
updated: "2013-11-02 12:28:41"
description: 
blogger:
    siteid: "32090046"
    postid: "932149362360686524"
    comments: "3"
categories: [python]
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

There are two bugs that prevent the python M2Crypto 0.21.1 package from being `pip install`ed from pypi in a virtualenv on Ubuntu 13.10.

* First bug [#696327](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=696327). libssl-dev is now multiarch and M2Crypto can't find the opensslconf.h header:

    ```
    SWIG/_evp.i:12: Error: Unable to find 'openssl/opensslconf.h'
    SWIG/_ec.i:7: Error: Unable to find 'openssl/opensslconf.h'
    error: command 'swig' failed with exit status 1
    ```

* Second bug [#637750](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=637750). If you do get M2Crypto to build by adding `/usr/include/x86_64-linux-gnu/` to [include_dirs](http://stackoverflow.com/a/19253719/1480205) in `~/.pydistutils.cfg`, SSLv2 has been disabled in openssl and M2Crypto fails to import:

    ```
    >>> import M2Crypto
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/home/ubuntu/ENV/local/lib/python2.7/site-packages/M2Crypto/__init__.py", line 22, in <module>
        import __m2crypto
    ImportError: /home/ubuntu/ENV/local/lib/python2.7/site-packages/M2Crypto/__m2crypto.so: undefined symbol: SSLv2_method
    ```

The Ubuntu python-m2crypto package has patches for both these, in `m2crypto_0.21.1-3ubuntu3.debian.tar.gz` available [here](http://packages.ubuntu.com/saucy/python-m2crypto).

These have been merged into the [debian git repository](http://anonscm.debian.org/gitweb/?p=collab-maint/m2crypto.git;a=blob;f=debian/patches/README.patches;h=32c21da601729c56ad18668d7a42b9f1856c8fd9;hb=HEAD), so the easiest route is to use pip's git support and install directly from there. The patches require you to specify your architecture using an environment variable so the full command for 64 bit would be:

```bash
DEB_HOST_MULTIARCH=x86_64-linux-gnu pip install "git+git://anonscm.debian.org/collab-maint/m2crypto.git@debian/0.21.1-3#egg=M2Crypto"
```
