---
title: "AWS VPN with Public Subnet"
layout: "post"
permalink: "/2016/01/aws-vpn-public-subnet.html"
tags: ["aws", "ipsec"]
---
This describes how to configure an ipsec VPN in an AWS VPC with a customer who does not allow RFC-1918 (private) IP addresses in the VPC subnet.

The basic idea is to expose a single host in the VPN using a `/32` subnet of the VPN public IP. We can restrict each client peer to a specific port on that host and use port forwarding to connect them to internal hosts on private subnets in the VPC. So we can support multiple clients, and each client sees only a single host (with a public IP) and can access a single client specific port on that host.

The following applies to Ubuntu 14.04 and [Strongswan](https://www.strongswan.org/) 5.1.2. For purposes of discussion we have two clients. All public IPs are invalid examples. First, clientA with peer public IP 165.{A}.22.101 and an internal host with public IP 165.{A}.22.102. We will restrict clientA to port 2575. Second, clientB with peer public IP 180.{B}.89.101 and an internal host with public IP 180.{B}.89.102. We will restrict clientB to port 2576.

We create an EC2 instance in a VPC with CIDR `10.0.0.0/16` and place it in a [public subnet](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Scenario1.html) with CIDR `10.0.0.0/24` and assign an EIP to it. Attach a [security group](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_SecurityGroups.html) to this instance that opens UDP ports 500 and 4500 to the internet - CIDR `0.0.0.0/0`. The public EIP we are assigned for this example is 52.{AWS}.113.223, and our private IP is 10.0.0.100.

First, we install Strongswan

```console
user@vpn$ sudo apt-get install strongswan
```

AWS allows you to associate a public IP ([EIP](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)) with an instance. This is implemented as a 1:1 NAT - the instance has a private IP in the VPC and is not aware of it's EIP. We are going to configure this hosts public EIP as the subnet for our peer. Normally the EIP 1:1 NAT rewrites packets destined for the EIP 52.{AWS}.113.223 to change the destination to our private IP 10.0.0.100. But packets tunneled through the VPN is encrypted and so the NAT cannot rewrite them, so we receive packets destined for EIP 52.{AWS}.113.223 and would normally route them back out. To work around this we will attach the EIP as an alias to `eth0`. So when packats arrive through the tunnel destined for 52.{AWS}.113.223 they will be accepted.

Create `/etc/network/interfaces.d/eth0:0.cfg` with:

```
auto eth0:0
iface eth0:0 inet static
name Ethernet alias public EIP
address 52.{AWS}.113.223
netmask 255.255.255.255
```

Then bring the interface up:

```console
user@vpn$ sudo ifup eth0:0
```

To automate provisioning a host, you can determine the public IP assigned to it by fetching an AWS metadata URL from the host itself:

```console
user@vpn$ curl http://169.254.169.254/latest/meta-data/public-ipv4
52.{AWS}.113.223
```

Now we can configure Strongswan. First disable NAT keep-alives, we don't need them for the 1:1 EIP NAT. Create `/etc/strongswan.d/charon-local.conf` with:

```
charon {
    keep_alive = 0
}
```

Next configure the base [ipsec config](https://wiki.strongswan.org/projects/strongswan/wiki/ConnSection). This contains the common shared configuration and will include separate client specific config files. We use our EIP as `leftid` and our VPC CIDR as `left`, so we will listen on our private IP. Create `/etc/ipsec.conf` with:

```
config setup

conn %default
    authby=psk
    ikelifetime=480m
    keylife=20m
    rekeymargin=3m
    keyingtries=1
    mobike=no
    keyexchange=ikev1
    dpdtimeout=900s
    auto=add
    type=tunnel

conn vpn-defaults
    left=10.0.0.0/16
    leftid=52.{AWS}.113.223

include /etc/ipsec.conf.d/*.conf
```

Mext create the client specific include files. Note that the first line in the include files must be a blank line or a comment.

Create `/etc/ipsec.conf.d/clientA.conf`:

```
# First line must be empty or comment
conn vpn-clientA
    leftsubnet=52.{AWS}.113.223/32[tcp/2575]
    right=165.{A}.22.101
    rightsubnet=165.{A}.22.102/32
    also=vpn-defaults
```

This restricts clientA to port 2575 on 52.{AWS}.113.223, and we configure clientA VPN peer IP and the single host we allow as `rightsubnet`.

Create `/etc/ipsec.conf.d/clientB.conf`:

```
# First line must be empty or comment
conn vpn-clientB
    leftsubnet=52.{AWS}.113.223/32[tcp/2576]
    right=180.{B}.89.101
    rightsubnet=180.{B}.89.102/32
    also=vpn-defaults
```

This restricts clientA to port 2576 on 52.{AWS}.113.223, and we configure clientB VPN peer IP and the single host we allow as `rightsubnet`.

Now generate shared secrets for each client:

```console
user@vpn$ dd if=/dev/random count=32 bs=1 2>/dev/null | base64
N/5y0EkgCFevc9uvJv1GsMG14BwbknCERoyJKZatoh0=
user@vpn$ dd if=/dev/random count=32 bs=1 2>/dev/null | base64
eb3W4WYXuHsnc0KVzg65G3XFQUD/I69/pYqFCYb5+iQ=
```

And configure the secrets files, owned by root and `chmod 0400` on all secrets files.

Create `/etc/ipsec.secrets`:

```
include /etc/ipsec.d/secrets/*.secret
```

Create `/etc/ipsec.d/secrets/clientA.secret`:

```
165.{A}.22.101 52.{AWS}.113.223 : PSK 0sN/5y0EkgCFevc9uvJv1GsMG14BwbknCERoyJKZatoh0=
```

Create `/etc/ipsec.d/secrets/clientB.secret`:

```
180.{B}.89.101 52.{AWS}.113.223 : PSK 0seb3W4WYXuHsnc0KVzg65G3XFQUD/I69/pYqFCYb5+iQ=
```

We specify the base64 encoded secret, prefixed with `0s`. The single space around the `:` is significant and required. For each secret we specify our public EIP and our client peers public IP.

Now we can bring up the VPN with each client:

```console
user@vpn$ sudo restart strongswan
user@vpn$ sudo ipsec up vpn-clientA
user@vpn$ sudo ipsec up vpn-clientB
user@vpn$ sudo ipsec statusall
Status of IKE charon daemon (strongSwan 5.1.2, Linux 3.13.0-76-generic, x86_64):
  uptime: 29 seconds, since Jan 25 14:58:53 2016
  malloc: sbrk 1486848, mmap 0, used 372928, free 1113920
  worker threads: 11 of 16 idle, 5/0/0/0 working, job queue: 0/0/0/0, scheduled: 6
  loaded plugins: charon test-vectors aes rc2 sha1 sha2 md4 md5 rdrand random nonce x509 revocation constraints pkcs1 pkcs7 pkcs8 pkcs12 pem openssl xcbc cmac hmac ctr ccm gcm attr kernel-netlink resolve socket-default stroke updown eap-identity addrblock
Listening IP addresses:
  10.0.0.100
  52.{AWS}.113.223
Connections:
vpn-defaults:  10.0.0.0/16...%any  IKEv1
vpn-defaults:   local:  [52.{AWS}.113.223] uses pre-shared key authentication
vpn-defaults:   remote: uses pre-shared key authentication
vpn-defaults:   child:  dynamic === dynamic TUNNEL
vpn-clientA:  10.0.0.0/16...165.{A}.22.101  IKEv1
vpn-clientA:   local:  [52.{AWS}.113.223] uses pre-shared key authentication
vpn-clientA:   remote: [165.{A}.22.101] uses pre-shared key authentication
vpn-clientA:   child:  52.{AWS}.113.223/32[tcp/2575] === 165.{A}.22.102/32 TUNNEL
vpn-clientB:  10.0.0.0/16...180.{B}.89.101  IKEv1
vpn-clientB:   local:  [52.{AWS}.113.223] uses pre-shared key authentication
vpn-clientB:   remote: [180.{B}.89.101] uses pre-shared key authentication
vpn-clientB:   child:  52.{AWS}.113.223/32[tcp/2576] === 180.{B}.89.102/32 TUNNEL
Security Associations (2 up, 0 connecting):
 vpn-clientA[2]: ESTABLISHED 10 seconds ago, 10.0.0.100[52.{AWS}.113.223]...165.{A}.22.101[165.{A}.22.101]
 vpn-clientA[2]: IKEv1 SPIs: 14af2c71cf57c3eb_i 1758db1b208b242f_r*, pre-shared key reauthentication in 7 hours
 vpn-clientA[2]: IKE proposal: AES_CBC_128/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_2048
 vpn-clientA{2}:  INSTALLED, TUNNEL, ESP in UDP SPIs: cab4c261_i c3ba251c_o
 vpn-clientA{2}:  AES_CBC_128/HMAC_SHA1_96, 0 bytes_i, 0 bytes_o, rekeying in 15 minutes
 vpn-clientA{2}:   52.{AWS}.113.223/32[tcp/2575] === 165.{A}.22.102/32
 vpn-clientB[1]: ESTABLISHED 17 seconds ago, 10.0.0.100[52.{AWS}.113.223]...180.{B}.89.101[180.{B}.89.101]
 vpn-clientB[1]: IKEv1 SPIs: 4b8896496ae4e206_i 0f6f62f73491847e_r*, pre-shared key reauthentication in 7 hours
 vpn-clientB[1]: IKE proposal: AES_CBC_128/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_2048
 vpn-clientB{1}:  INSTALLED, TUNNEL, ESP in UDP SPIs: ce665f80_i c2e8c988_o
 vpn-clientB{1}:  AES_CBC_128/HMAC_SHA1_96, 0 bytes_i, 0 bytes_o, rekeying in 13 minutes
 vpn-clientB{1}:   52.{AWS}.113.223/32[tcp/2576] === 180.{B}.89.102/32
```

We can test connectivity by listening to a client specific port with netcat and having that client attempt to connect to it over the vpn.

On the vpn:

```console
user@vpn$ nc -k -v -l 2575
```

On clientA:

```console
user@165.{A}.22.102$ nc 52.{AWS}.113.223 2575
```

Now we can install and configure [rinetd](http://www.boutell.com/rinetd/) which we can use to redirect client ports 2575 and 2576 to listening hosts on internal subnets.

```console
user@vpn$ sudo apt-get install rinetd
```

Create `/etc/rinetd.conf` with:

```
# bindadress    bindport  connectaddress  connectport
0.0.0.0 2575 internal-host 2575
0.0.0.0 2576 internal-host 2576

# logging information
logfile /var/log/rinetd.log
```

This forwards connections to ports 2575 and 2576 on this host to a host named `internal-host` on the private subnet. That host would need a security group opening those ports, or you could also send traffic over an ipsec transport as described in [another post](/2015/03/ipsec-private-subnet.html). In that case, you would configure Strongswan to initiate an ipsec transport with Racoon on `internal-host`.
