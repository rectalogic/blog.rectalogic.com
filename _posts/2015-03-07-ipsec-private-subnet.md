---
title: "IPsec Private Subnet"
layout: "post"
permalink: "/2015/03/ipsec-private-subnet.html"
---
Sometimes you want to secure local traffic in a private subnet for compliance reasons, e.g. HIPAA requires data in-transit to be encrypted. This can be done at the application level if the application supports SSL, but it can also be done independent of the application using IPsec transport layer encryption.

In this example we have an AWS [VPC](http://aws.amazon.com/vpc/) with three Ubuntu 14.04 database instances in a private subnet running a mongodb replicaset with three members. We want to encrypt all mongodb traffic between replicaset members and between other client instances in the VPC and the mongodb instances - and we won't use mongodb SSL support.

The [mad-hacking](http://www.mad-hacking.net/documentation/linux/networking/ipsec/index.xml) site has a good discussion of IPsec, racoon and setkey.

First install ipsec-tools and racoon on each instance: `sudo apt-get install ipsec-tools racoon`.

Create a pre-shared key file for use with ISAKMP. We are using the wildcard `*` to match all hosts, so all instances are using the same key. Generate this file once and install the same file on all instances:

Generate the file with a random key:

```shell-session
user@host$ echo "* " $(openssl rand -base64 48) > psk.txt
user@host$ cat psk.txt
*  /zatck0CNzYad4fGw4ZV9G1R8b93pdBzCc3I7iXh2zzvediol0yBR8F8wCHKryXE
```

Install the same file on each instance:

```shell-session
user@host$ sudo cp psk.txt /etc/racoon/psk.txt
user@host$ sudo chmod 0400 /etc/racoon/psk.txt
user@host$ sudo chown root.root /etc/racoon/psk.txt
```

Next create a racoon configuration in `/etc/racoon/racoon.conf` on each instance. See the [racoon.conf](http://manpages.ubuntu.com/manpages/trusty/man5/racoon.conf.5.html) man page for details, this also discusses the pre-shared key file format above.

We don't specify `my_identifier` in the `remote` section, so our private IP address is used as the identifier. You can change `log debug;` to `log info;` once things are working.

`/etc/racoon/racoon.conf`

```
log debug;

path pre_shared_key "/etc/racoon/psk.txt";

complex_bundle on;

listen {
    adminsock "/var/run/racoon/racoon.sock";
}

remote anonymous
{
    exchange_mode base;
    passive off;
    nat_traversal off;
    dpd_delay 30;

    proposal
    {
        encryption_algorithm aes;
        hash_algorithm sha1;
        authentication_method pre_shared_key;
        dh_group modp2048;
        lifetime time 24 hour;
    }
}

sainfo anonymous
{
    encryption_algorithm aes, 3des;
    authentication_algorithm hmac_sha512, hmac_sha384, hmac_sha256, hmac_sha1;
    compression_algorithm deflate;
    pfs_group modp1024;
    lifetime time 12 hour;
}
```

Next, create a security policy that will require mongodb traffic (tcp port 27017) to be encrypted. Copy this file to `/etc/ipsec-tools.d/mongodb.conf` on each instance. The sample file uses a src and dst range that encompasses our entire VPC CIDR block - so all traffic on tcp 27017 between any hosts in our VPC will be encrypted.

`/etc/ipsec-tools.d/mongodb.conf`

```
spdadd 10.0.0.0/16 10.0.0.0/16[27017] tcp -P in ipsec esp/transport//require;
spdadd 10.0.0.0/16[27017] 10.0.0.0/16 tcp -P out ipsec esp/transport//require;
spdadd 10.0.0.0/16[27017] 10.0.0.0/16 tcp -P in ipsec esp/transport//require;
spdadd 10.0.0.0/16 10.0.0.0/16[27017] tcp -P out ipsec esp/transport//require;
```

Now you need to allow ISAKMP to talk on udp port 500, and enable IP protcol 50 (ESP). This is an example [CloudFormation](http://aws.amazon.com/documentation/cloudformation/) configuration for an `IPsecSecurityGroup` - apply this security group to all your instances in the VPC.

**AWS CloudFormation Security Group**

```json
{
    "Resources": {
        "VPC": {
            "Type": "AWS::EC2::VPC",
            "Properties": {
                "CidrBlock": "10.0.0.0/16",
                "InstanceTenancy": "default"
            }
        },
       "IPsecSecurityGroup": {
            "Type": "AWS::EC2::SecurityGroup",
            "Properties": {
                "GroupDescription": "Enable IPsec ISAKMP UDP 500 and ESP protocol 50",
                "VpcId": {
                    "Ref": "VPC"
                }
            }
        },
        "IPsecSecurityGroupIngressISAKMP": {
            "Type": "AWS::EC2::SecurityGroupIngress",
            "Properties": {
                "GroupId": {
                    "Ref": "IPsecSecurityGroup"
                },
                "SourceSecurityGroupId": {
                    "Ref": "IPsecSecurityGroup"
                },
                "IpProtocol": "udp",
                "FromPort": "500",
                "ToPort": "500"
            }
        },
        "IPsecSecurityGroupIngressESP": {
            "Type": "AWS::EC2::SecurityGroupIngress",
            "Properties": {
                "GroupId": {
                    "Ref": "IPsecSecurityGroup"
                },
                "SourceSecurityGroupId": {
                    "Ref": "IPsecSecurityGroup"
                },
                "IpProtocol": "50"
            }
        }
    }
}
```

Finally, on each instance restart setkey - this will flush any existing SA and SPD kernel entries and load the mongodb security policy:

```shell-session
user@host$ sudo service setkey restart
 * Flushing IPsec SA/SP database:                                        [ OK ]
 * Loading IPsec SA/SP database:
                                                                         [ OK ]
done.
```

Then restart racoon on each instance:

```shell-session
user@host$ sudo service racoon restart
 * Restarting IKE (ISAKMP/Oakley) server racoon                          [ OK ]
```

Right now there are no ISAKMP SAs and no IPsec SAs:

```shell-session
user@host$ sudo racoonctl show-sa isakmp
Destination            Cookies                           Created
user@host$ sudo racoonctl show-sa ipsec
No SAD entries.
```

If you attempt to establish a connection between two instances on port 27017 (in this case attempting to netcat to another instance on port 27017), this will negotiate a new security association.  You can troubleshoot by looking for racoon error messages in `/var/log/syslog`.

```shell-session
user@host$ netcat dev2 27017
user@host$ sudo racoonctl show-sa isakmp
Destination            Cookies                           Created
10.0.1.241.500         6cfa102c16746d63:34488f0a554ed72f 2015-03-06 21:59:06
user@host$ sudo racoonctl show-sa ipsec
10.0.1.13 10.0.1.241
    esp mode=transport spi=163029363(0x09b7a173) reqid=0(0x00000000)
    E: aes-cbc  7575d5cc ceb2f132 d3ca09c1 08bc1b0c
    A: hmac-sha512  b7c5d5e8 18a18929 ec676e49 cdf45d23 a00ca0ba c7fcb7d4 200ad3b8 8dc4271c
 ed6454f4 c4d8604a 3f9ecec6 e5093763 39e669fc 37d4f989 b1f135be 9348171e
    seq=0x00000000 replay=4 flags=0x00000000 state=mature
    created: Mar  6 21:59:07 2015   current: Mar  6 21:59:17 2015
    diff: 10(s) hard: 43200(s)  soft: 34560(s)
    last: Mar  6 21:59:09 2015  hard: 0(s)  soft: 0(s)
    current: 40(bytes)  hard: 0(bytes)  soft: 0(bytes)
    allocated: 1    hard: 0 soft: 0
    sadb_seq=1 pid=20603 refcnt=0
10.0.1.241 10.0.1.13
    esp mode=transport spi=187807444(0x0b31b6d4) reqid=0(0x00000000)
    E: aes-cbc  79d55f95 3bc0377b 0f7e5828 7a849525
    A: hmac-sha512  26068868 78cd42f7 5c10521b e55ed143 118b538e 6903c122 2463de8c 4fdebba6
 941ad0a6 8dd1b09e 6b0fa126 d0c9a7d8 929c2ce7 e3cfe861 cacfe3b4 8a9c1c28
    seq=0x00000000 replay=4 flags=0x00000000 state=mature
    created: Mar  6 21:59:07 2015   current: Mar  6 21:59:17 2015
    diff: 10(s) hard: 43200(s)  soft: 34560(s)
    last: Mar  6 21:59:09 2015  hard: 0(s)  soft: 0(s)
    current: 20(bytes)  hard: 0(bytes)  soft: 0(bytes)
    allocated: 1    hard: 0 soft: 0
    sadb_seq=0 pid=20603 refcnt=0
```

Now if you install mongodb and configure a replicaset on your three hosts, they will communicate with each other over an IPsec encrypted transport. You can sniff to verify ESP traffic is being used:

```shell-session
user@host$ sudo tcpdump -vv -i eth0 esp
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
22:03:21.188112 IP (tos 0x0, ttl 64, id 12303, offset 0, flags [DF], proto ESP (50), length 236)
    10.0.1.13 > 10.0.1.241: ESP(spi=0x09b7a173,seq=0x3b), length 216
22:03:21.188698 IP (tos 0x0, ttl 64, id 54545, offset 0, flags [DF], proto ESP (50), length 252)
    10.0.1.241 > 10.0.1.13: ESP(spi=0x0b31b6d4,seq=0x39), length 232
...
```