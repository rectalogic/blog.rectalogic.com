---
title: "Virtualizing Mac OS X Leopard Client"
layout: "post"
permalink: "/2008/08/virtualizing-mac-os-x-leopard-client.html"
uuid: "1322539730533180569"
guid: "tag:blogger.com,1999:blog-32090046.post-1322539730533180569"
date: "2008-08-05 03:23:00"
updated: "2011-09-28 17:39:08"
description: 
blogger:
    siteid: "32090046"
    postid: "1322539730533180569"
    comments: "136"
categories: 
author: 
    name: "rectalogic"
    url: "undefined?rel=author"
    image: "http://img2.blogblog.com/img/b16-rounded.gif"
---

[VMWare Fusion 2.0 beta2](http://communities.vmware.com/community/beta/fusion) supports virtualizing Mac OS X Server as a guest OS. If you try to install a Leopard Client guest, you get an error:
> The guest operating system is not Mac OS X Server.

However, if you create an ISO/CDR image from your Leopard install DVD, mount it then do

```bash
touch "/Volumes/Mac OS X Install DVD/System/Library/CoreServices/ServerVersion.plist"
```

then unmount it, you can now use that image to install Leopard Client into VMWare with no complaints.  After you install, reboot VMWare from the install DVD ISO again, run Terminal and

```bash
touch "/Volumes/Macintosh HD/System/Library/CoreServices/ServerVersion.plist"
```

then reboot from the HD.  This probably violates your license agreement so don't do it, I certainly wouldn't.

**Update:** You can automate the deletion and creation of the ServerVersion.plist file using a LaunchDaemon. Put the following xml in a new file `/Library/LaunchDaemons/com.rectalogic.vmware.plist`:

```XML
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.rectalogic.vmware</string>
    <key>ProgramArguments</key>
    <array>
            <string>/bin/bash</string>
            <string>-c</string>
            <string>/bin/rm -f /System/Library/CoreServices/ServerVersion.plist; trap "/usr/bin/touch /System/Library/CoreServices/ServerVersion.plist; exit" SIGINT SIGTERM SIGHUP; sleep 999999 & wait $!</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

Then run:

```bash
sudo launchctl load /Library/LaunchDaemons/com.rectalogic.vmware.plist
```

Now when you login ServerVersion.plist will be deleted, when you shutdown it will be recreated ready for the next reboot.

**Alternative approach:**  An alternative approach discussed in the comments is to hack VMWare to disable the check for server.

```bash
sudo bash
cd "/Library/Application Support/VMware Fusion/isoimages"
mkdir original
mv darwin.iso tools-key.pub *.sig original
perl -n -p -e 's/ServerVersion.plist/SystemVersion.plist/g' > darwin.iso
openssl genrsa -out tools-priv.pem 2048
openssl rsa -in tools-priv.pem -pubout -out tools-key.pub
openssl dgst -sha1 -sign tools-priv.pem > darwin.iso.sig
for A in *.iso ; do openssl dgst -sha1 -sign tools-priv.pem < $A > $A.sig ; done
exit
```

**VMWare Fusion 3.0:** Fusion 3.0 uses EFI instead of BIOS by default. After creating a new VM and before booting it from the install DVD/ISO, edit the *.vmx file and remove/comment out the `firmware="efi"` line. VMWare will then use the hacked boot image from darwin.iso.

**VMWare Fusion 4.0:** Fusion 4.0 is similar to 3.0, but the path in the script above must be changed from `/Library/Application Support/VMware Fusion/isoimages` to `/Applications/VMware Fusion.app/Contents/Library/isoimages`. Each time you boot the guest you will get an error dialog **No operating system was found**. Click **OK**. You may then get an error dialog **Your Mac OS guest is using this CD-ROM device**. Click **Cancel**. Now you should have a black screen with **No operating system found** displayed, click on the window and hit **Return** to boot.
