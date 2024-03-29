---
layout: post
title:  "Linux Networking Bridge"
author: Joachim Wiberg
date:   2020-03-25 00:20:42 +0100
tag: howto
---

This is the second post in a series of posts showing how to set up
networking in Linux using low-level tools.

It's time to talk about bridging (switching) and VLANs.

<!-- more -->

## Bridging, or Switching

The [fist post]({% post_url 2020-03-24-linux-networking-intro %})
introduced LANs and broadcast domains.  An Ethernet bridge, or more
commonly, a switch, connects multiple networks segments into a common
broadcast domain.  If you are interested in this, see [the Wikipedia
page on bridging](https://en.wikipedia.org/wiki/Bridging_(networking))
for details.

In Linux we can create a software defined switch by adding multiple
network interfaces (NICs) to a PC and then connect them to the bridge
module.  In this setup these interfaces are called ports, and we don't
set IP addresses on them.  Instead, we do that on the bridge, and on
interfaces on top of the bridge.

                        br0          <---- bridge interface
                     ____|____
                    |#_#_#_#_#|      <---- bridge
                    /  |   |  \
                eth0 eth1 eth2 eth3  <---- ports


## Virtual LANs, VLANs

To kick things up a notch we need to introduce one more concept before
moving on -- [VLANs](https://en.wikipedia.org/wiki/IEEE_802.1Q)!

A VLAN, or *virtual* LAN, is one of the true corner stones in most
network setups, and as such is really deserves a blog post of its own.

However, for the purpose of this post, consider VLANs a way for us to
*group ports* in separate broadcast domains.  I.e., isolate certain end
devices from each other; e.g., an office network from a process control
network.

                          br0
                     ______|______
                    |#_#_#_#_#_#_#|
                    /  |   :  |    \ 
                eth0 eth1  :  eth2 eth3
                           :
                  VLAN 1   :    VLAN 2

Here we have configured the bridge (switch) to assign ports eth0 and
eth1 to VLAN 1, and eth2 and eth3 to VLAN 2.  Ports in each VLAN can
only communicate with each other, the bridge ensures a true separation
between both VLANs.

If a device on port eth0 (member of VLAN 1) wants to communicate with a
device on port eth3 (member of VLAN 2) it must be routed somehow.  For
this to work we must either connect a router to ports eth1 and eth2, or
let interface br0 be a member of both VLANs.

> A port that is member of more than one VLAN is often referred to as a
> *trunk port*, and a port facing an end device is called *access port*.

Port VLAN memberships can be *tagged* or *untagged*.  A tagged port is
usually a trunk port, and an untagged port is usually an access port.
There are always exceptions to these rules, but for most cases this is a
good starting point.

To route traffic between VLAN 1 and VLAN 2 we create the following
setup (it's starting to look a bit crazy now):

    IP: 192.168.1.1                     IP: 192.168.2.1
                      br0.1     br0.2
                           \   /
                            br0
                       ______|______
                      |#_#_#_#_#_#_#|
                      /  |   :  |    \ 
                  eth0 eth1  :  eth2 eth3
                             :
                    VLAN 1   :    VLAN 2

Since `br0` now is a tagged member of both VLANs we need to create VLAN
interfaces on top of it to be able to set IP addresses.  These are the
gateway addresses each end device will use in their IP network setup.

That is basically it, remember to enable IP forwarding ... now let's get
hands-on with the command line!

> In the next section we use the names `vlan1` and `vlan2` instead of
> `br0.1` and `br0.2`, respectively.  The naming is not only create
> confusion, but to a) show that any name can be used, and b) simplify
> and follow the terminology used in Westermo WeOS.


## Creating a Bridge in Linux

There are actually two variants of the standard bridge in mainline
Linux; old-style and new-style.  The latter, which we will focus on in
this blog post, has native support for VLAN filtering.

    # ip link add br0 type bridge
    # ip link set br0 type bridge vlan_filtering 1

> **Note:** recent versions of Debian based systems, like Ubuntu, have
> enabled bridge firewalling by default.  This may completely disable
> all or some forwarding of traffic on bridges.  Causing a lot of head
> scratching!  See [Bridge Forwarding Problem][] for a fix!

Now, add a couple of ports to the bridge:

    # ip link set eth0 master br0
    # ip link set eth1 master br0

To see the ports we use the [bridge(8)][] command, which is also part
of the iproute2 tool suite:

    # bridge link
    2: eth0: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 100 
    4: eth1: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 4 

To see the default VLAN assignments of ports:

    # bridge vlan show
    port    vlan ids
    eth0     1 PVID Egress Untagged
    eth1     1 PVID Egress Untagged
    br0      1 PVID Egress Untagged

So these ports look OK, the default VLAN ID assigned to ports is 1.
Lets add the other two, but now we need to tell the bridge to use VLAN
ID 2 instead.  We also set the `pvid` and `untagged` flags since we
want to treat these ports as access ports (untagged), and assign their
default VLAN (ID 2) on ingress (pvid).  Remember to remove from their
default VLAN (ID 1) as well:

    # ip link set eth2 master br0
    # ip link set eth3 master br0
    # bridge vlan add vid 2 dev eth2 pvid untagged
    # bridge vlan add vid 2 dev eth3 pvid untagged
    # bridge vlan del vid 1 dev eth2
    # bridge vlan del vid 1 dev eth3

To see static and learned MAC addresses (c.f. the `arp` command):

    # bridge fdb show
    00:80:e1:42:55:a3 dev eth0 vlan 1 master br0 permanent
    00:80:e1:42:55:a3 dev eth0 master br0 permanent
    33:33:00:00:00:01 dev eth0 self permanent
    00:e0:4c:68:03:06 dev eth1 vlan 1 master br0 permanent
    00:e0:4c:68:03:06 dev eth1 master br0 permanent
    33:33:00:00:00:01 dev eth1 self permanent
	...
    33:33:00:00:00:01 dev br0 self permanent

In our use-case we have two different VLANs, so we need to change the
bridge port itself to be a *tagged* VLAN member, otherwise we cannot
distinguish between frames on different VLANs and thus cannot set up
our VLAN interfaces on top, like this:

        vlan1     vlan2      Layer-3 :: IP Networking
             \   /           -------------------------------
              br0
         ______|_______      Layer-2 :: Switching
        [#_#_#_#_#_#_#] 
        /  |   :  |    \     -------------------------------
    eth0 eth1  : eth2 eth3   Layer-1 :: Link layer
               :
      VLAN 1   :    VLAN 2

Let's change br0 to be a tagged member of VLAN 1 and 2:

    # bridge vlan add vid 1 dev br0 self
    # bridge vlan add vid 2 dev br0 self
    # bridge vlan show
    port    vlan ids
    eth0     1 PVID Egress Untagged
    eth1     1 PVID Egress Untagged
    eth2     2 PVID Egress Untagged
    eth3     2 PVID Egress Untagged
    br0      1
             2

Now we add our VLAN interface on top of `br0` so we can communicate with
the outside world.  Some prefer naming VLAN interfaces `br0.1`, but here
we use `vlan1` since we will only use one bridge:

    # ip link add name vlan1 link br0 type vlan id 1
    # ip addr add 192.168.1.1/24 dev vlan1
    # ip link add name vlan2 link br0 type vlan id 2
    # ip addr add 192.168.2.1/24 dev vlan2

Bring everything up by taking up the bridge and its ports:

    # ip link set eth0 up
    # ip link set eth1 up
    # ip link set eth2 up
    # ip link set eth3 up
    # ip link set br0 up
    # ip link set vlan1 up
    # ip link set vlan2 up

This is a good time to have a look at the available interfaces:

	# ip -brief link show
    lo        UNKNOWN  00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
    eth0      UP       00:80:e1:42:55:a0 <NO-CARRIER,BROADCAST,MULTICAST,UP>
    eth1      UP       00:80:e1:42:55:a1 <BROADCAST,MULTICAST,UP,LOWER_UP>
    eth2      UP       00:80:e1:42:55:a2 <NO-CARRIER,BROADCAST,MULTICAST,UP>
    eth3      UP       00:80:e1:42:55:a3 <BROADCAST,MULTICAST,UP,LOWER_UP>
    br0       UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP>
    vlan1@br0 UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP> 
    vlan2@br0 UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP>

As you can see, the `vlan1` interface is created on top of `br0`,
`vlan1@br0`.  The addresses of all interfaces can be inspected with the
`ip address` command.  For a quick overview, use the `-brief` switch:

    # ip -br addr show
    lo               UNKNOWN        127.0.0.1/8
    eth0             UP             
    eth1             UP             
    eth2             UP             
    eth3             UP             
    br0              UP             
    vlan1@br0        UP             192.168.1.1/24
    vlan2@br0        UP             192.168.2.1/24

Here we have automatically configured IPv6 addresses on eth1 and br0,
this should be disabled since IP addresses in a our bridge setup should
only be set on the VLAN interfaces.


## Summary and More

In this post we covered the theory of Ethernet bridges and VLANs, and
then proceeded to provide an example of how to set this up a single
bridge up in Linux.

But wait, what if we want to connect two separate bridges, on two PCs,
with multiple VLANs on each?  Let's extend the image used previously,
and add a syntax for denoting VLAN memberships: *1U* means untagged
member of VLAN 1, *2U* means untagged in VLAN 2, and *1T* means tagged
member of VLAN 1, etc.

      vlan1     vlan2              vlan1     vlan2
           \   /                        \   /
            br0  1T,2T                   br0  1T,2T,3T
         ____|____                    ____|__________
        [#_#_#_#_#]                  [#_#_#_#_#_#_#_#]
        /  |      \                  /  |   |   \    \
    eth2  eth1     eth0----------eth0 eth1 eth2  eth3 eth4
     2U    1U             1T,2T        1U   2U    3U   3U

The image shows two devices with one bridge each.  The right-hand bridge
has more ports and VLANs, but they are interconnected using port eth0 on
each bridge.  This shared link, VLAN "trunk" (see above), serves as the
backbone for this network.

> Notice how VLAN 3 only exists on the right-hand bridge, both bridges
> filter traffic going out and coming in on the trunk from port eth0, to
> prevent VLAN 3 from reaching beyond its boundary (port eth3 and eth4).


## EOF

Future posts will cover how the Linux bridge can be used with single
board computers that support switching in hardware, i.e., offloading of
the otherwise CPU intensive parts.

Feel free to contact Westermo for more information, help designing your
network, and hands on training on our products.

Visit <https://www.westermo.com>



[bridge(8)]: http://man7.org/linux/man-pages/man8/bridge.8.html
[Bridge Forwarding Problem]: /2021/06/24/bridge-forwarding-problem/
