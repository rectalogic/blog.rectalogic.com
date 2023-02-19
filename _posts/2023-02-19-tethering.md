---
title: "Tethering"
layout: "post"
permalink: "/2023/02/tethering.html"
---

In iOS when you enable the Personal Hotspot, it creates a separate
`bridge100` interface.
So your tethered traffic goes out over this interface instead of the one your device traffic uses.
So it counts against your tethered traffic instead of device traffic.

There is a way to make the tethered traffic use the same outbound interface
that device traffic uses.
This is a walk-through of how to do that.

First, install [a-shell](https://holzschu.github.io/a-Shell_iOS/) on iOS.
Run it and enter `pip install pvpn` (see [pvpn](https://pypi.org/project/pvpn/)).

Then enter `pvpn -wg 9000`. You should see something like this:
![a-shell-wireguard.jpg]({{ site.baseurl }}/images/a-shell-wireguard.jpg)
Copy the `PublicKey` it prints.

Next install the macOS [WireGuard client](https://apps.apple.com/us/app/wireguard/id1451685025?mt=12)

Choose *Add Empty Tunnel...*, name it `pvpn` and configure similar to this,
where `[Peer]` `PublicKey` is the key the server printed above:
![wireguard-config.png]({{ site.baseurl }}/images/wireguard-config.png)

Activate the WireGuard tunnel and macOS traffic should not be over the VPN
running on the iOS device.

You can confirm both the iOS device and the macOS client have the same public IP by visiting http://ifconfig.co/ in Safari on iOS and in a browser on macOS.
