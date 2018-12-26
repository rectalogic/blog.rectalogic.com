---
title: "Haystack Arq 5 APFS Snapshot Backup"
layout: "post"
permalink: "/2018/12/arq5-apfs-snapshot-backup.html"
tags: ["arq", "backup", "APFS"]
---

[Arq 5 backup](https://www.arqbackup.com) on macOS does not automatically create and back up from an [APFS snapshot](https://en.wikipedia.org/wiki/Apple_File_System#Snapshots).

By configuring Arq preferences with before and after backup scripts, you can take an APFS snapshot, mount it and perform the backup from the snapshot, then unmount when finished. This way the backup is point-in-time consistent.

![preferences.png]({{ site.baseurl }}/images/arq/preferences.png)

`snapshot-before.sh`
```bash
#!/usr/bin/env bash

diskutil unmount /tmp/snapshot
set -o errexit

tmutil localsnapshot
SNAPSHOT=$(tmutil listlocalsnapshots / | tail -n 1)
mkdir -p /tmp/snapshot
mount -t apfs -r -o -s=$SNAPSHOT / /tmp/snapshot
```

`snapshot-after.sh`
```bash
#!/usr/bin/env bash

set -o errexit

diskutil unmount /tmp/snapshot
```

Run `snapshot-before.sh` manually and then in Arq select your home directory inside the snapshot as the directory to back up (`/tmp/snapshot/Users/XXX`).