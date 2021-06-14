---
layout: post
title:  "Tools of the Trade :: ping & tcpdump"
author: Joachim Wiberg
tags:
  - howto
  - tools
date:   2021-06-12 06:16:23 +0100
---

Continuing the series with another set of really useful tools to know:
[ping(8)](https://www.man7.org/linux/man-pages/man8/ping.8.html) and
[tcpdump(8)](https://www.man7.org/linux/man-pages/man8/tcpdump.8.html).
They can be used to localize issues in just about any network setup.

Ping is capable of generating unicast, multicast, and broadcast traffic.
While tcpdump is capable of capturing, formatting, *and most importantly
filtering* all types of traffic.

<!-- more -->

Recall the last picture from [Linux Networking Bridge][]:

```
  vlan1     vlan2              vlan1     vlan2
       \   /                        \   /
        br0  1T,2T                   br0  1T,2T,3T
     ____|____                    ____|__________
    [#_#_#_#_#]                  [#_#_#_#_#_#_#_#]
    /  |      \                  /  |   |   \    \
eth2  eth1     eth0----------eth0 eth1 eth2  eth3 eth4
 2U    1U             1T,2T        1U   2U    3U   3U
 |     |                           |    |     |    |
 |     |                           |    |     |    |
eth0  eth0                        eth0 eth0  eth0 eth0
ED1   ED2                         ED3  ED4   ED5  ED6
```
*Figure 1: All end-devices (ED) have a single interface `eth0`*

Much can go wrong setting that up.  Not just with the standard Linux
bridge, but also with any HW offloading switch, and their respective
drivers.   To test the setup we can use our friends ping and tcpdump.
The following subnets and IP addresses are used:

**Subnets:**

  - VLAN 1: 192.168.1.0/24
  - VLAN 2: 192.168.2.0/24

**Left:**

  - vlan1: 192.168.1.1
  - vlan2: 192.168.2.1
  - ED1: 192.168.2.11
  - ED2: 192.168.1.11

**Right:**

  - vlan1: 192.168.1.2
  - vlan2: 192.168.2.2
  - ED3: 192.168.1.22
  - ED4: 192.168.2.22
  - ED5: 192.168.3.33
  - ED6: 192.168.3.34


Connectivity Between 2.11 and 2.22
----------------------------------

We start by verifying connectivity between ED1 on the left and ED4 on
the right.  They should both be untagged members in VLAN 2, and the VLAN
trunk between the bridges should carry the same VLAN tagged.  From ED1:

    root@ed1:~# ping 192.168.2.22

If we don't get a reply we can check with tcpdump on ED4:

    root@ed4:~# tcpdump -lni eth0 icnmp

Dead silence.  So we go back to ED1 and change to a broadcast ping, this
should reach everyone connected to VLAN 2.  We check this with tcpdump
on all other ports.  First we check to see if we see anything on the
left bridge's VLAN 2:

    root@ed1:~# ping 192.168.2.255
    root@left:~# tcpdump -lni vlan2 icmp

Here we see the ICMP traffic, so we can move on to check if we get the
same traffic on the right-hand bridge as well:

    root@right:~# tcpdump -lni vlan2 icmp

Yup, so what's wrong here?  Verify the VLAN membership on the bridge:

    root@right:~# bridge vlan
	 
There we have it!  Ports `eth1` and `eth2` had been mixed up in their
VLAN assignments!


Deep Dive in the Stack
----------------------

Now that we've covered a basic troubleshooting case, let's dive into the
various layers in the networking stack of one of the bridging devices.

```
    IP: 192.168.1.1   vlan1     vlan2   IP: 192.168.2.1
                           \   /
                            br0
                       ______|______
                      |#_#_#_#_#_#_#|
                      /  |   :  |    \ 
                  eth0 eth1  :  eth2 eth3
                             :
                    VLAN 1   :    VLAN 2
```

We use the same basic tools, inject ICMP traffic with ping on one port
and use tcpdump to see where it ends up.  Here we'll use broadcast from
an "end-device" attached to eth0:

```
    IP: 192.168.1.1   vlan1     vlan2   IP: 192.168.2.1
                           \   /
                            br0
NS1: 192.168.1.10      ______|______
--------.             |#_#_#_#_#_#_#|
lo      :             /  |   :  |    \ 
eth0    :         eth0 eth1  :  eth2 eth3
    `-------------'          :
--------'           VLAN 1   :    VLAN 2
```

The "end-device" is a network namespace on a dedicated device, with a
dedicated network card, or a VETH pair with one end in the namespace,
and the other attached to the bridge.  The latter is useful for testing
on the same device where the bridge runs.

    root@ns1:~# ping 192.168.1.255

On the system itself we can start by running tcpdump at the bottom, the
interface connected to the bridge.  This should work regardless if the
system has bridging (switch) offloading to an underlying hardware, since
broadcast is forwarded to all hosts on a LAN.

    root@system:~# tcpdump -lni eth0 icmp

Here we can see ... we can add `-e` to get the Ethernet header as well:

    root@system:~# tcpdump -elni eth0 icmp



[Linux Networking Bridge]: /2020/03/25/linux-networking-bridge/
