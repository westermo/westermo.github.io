---
layout: post
title:  "Linux Networking Bridge"
date:   2020-03-25 00:20:42 +0100
categories: howto
---

This is the second post in a series of posts showing how to set up
networking in Linux using low-level tools.

It's time to talk about bridging (switching) and VLANs.


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

    IP: 192.168.1.1                     IP: 192.169.2.1
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
Linux; old-style and new-style.  The latter, which we will focus on,
has native support for VLAN filtering.

    # ip link add br0 type bridge
    # ip link set br0 type bridge vlan_filtering 1

Now, add a couple of ports to the bridge:

    # ip link set eth0 master br0
    # ip link set eth1 master br0

To see the ports:

    # bridge link
    2: eth0: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 100 
    4: eth1: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 4 

To see the default VLAN assignments of ports:

    # bridge vlan show
    port    vlan ids
    eth0     1 PVID Egress Untagged
    
    eth1     1 PVID Egress Untagged
    
    br0      1 PVID Egress Untagged

To see static and learned MAC addresses (c.f. the `arp` command):

    # bridge fdb show
    00:80:e1:42:55:a3 dev eth0 vlan 1 master br0 permanent
    00:80:e1:42:55:a3 dev eth0 master br0 permanent
    33:33:00:00:00:01 dev eth0 self permanent
    00:e0:4c:68:03:06 dev eth1 vlan 1 master br0 permanent
    00:e0:4c:68:03:06 dev eth1 master br0 permanent
    33:33:00:00:00:01 dev eth1 self permanent
    33:33:00:00:00:01 dev br0 self permanent

- [bridge(8)](http://man7.org/linux/man-pages/man8/bridge.8.html)

As you could see above, all ports are by default added as untagged
members of VLAN 1, and that including the bridge itself.  For our
use-case we want the bridge itself to be a *tagged* member, so we
can have multiple VLAN interfaces on top, like this:

       vlan1     vlan2      Layer-3 :: IP Networking
            \   /        -------------------------------
             br0
          ____|____         Layer-2 :: Switching
         [#_#_#_#_#] 
         /    |    \     -------------------------------
     eth0   eth1    eth2    Layer-1 :: Link layer
	
The image shows a second VLAN interface and a third port that are not
part of this HowTo.  See below for a discussion of Advanced VLANs.

Let's change br0 to be a tagged member of VLAN 1:

    # bridge vlan add vid 1 dev br0 self
    # bridge vlan show
    port    vlan ids
    eth0     1 PVID Egress Untagged
    
    eth1     1 PVID Egress Untagged
    
    br0      1

Now we add a VLAN interface on top of `br0` so we can communicate with
the outside world.  Many prefer naming VLAN interfaces `br0.1`, but we
use `vlan1` instead since we will only use one bridge:

    # ip link add name vlan1 link br0 type vlan id 1
    # ip addr add 192.168.2.200/24 dev vlan1

Bring everything up by taking up the bridge and its ports:

    # ip link set eth0 up
    # ip link set eth1 up
    # ip link set br0 up

Add a default route to your local gateway:

    # ip route add default via 192.168.2.42

This is a good time to have a look at the available interfaces:

	# ip link show
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master br0 state DOWN mode DEFAULT group default qlen 1000
        link/ether 00:80:e1:42:55:a3 brd ff:ff:ff:ff:ff:ff
    4: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br0 state UP mode DEFAULT group default qlen 1000
        link/ether 00:e0:4c:68:03:06 brd ff:ff:ff:ff:ff:ff
    5: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
        link/ether 00:80:e1:42:55:a3 brd ff:ff:ff:ff:ff:ff
    6: vlan1@br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
        link/ether 00:80:e1:42:55:a3 brd ff:ff:ff:ff:ff:ff

As you can see, the `vlan1` interface is created on top of `br0`,
`vlan1@br0`.  The addresses of all interfaces can be inspected with the
`ip address` command.  For a quick overview, use the `-brief` switch:

    # ip -br addr show
    lo               UNKNOWN        127.0.0.1/8 ::1/128 
    eth0             DOWN           
    eth1             UP             fe80::2e0:4cff:fe68:306/64 
    br0              UP             fe80::280:e1ff:fe42:55a3/64 
    vlan1@br0        UP             192.168.2.200/24 fe80::280:e1ff:fe42:55a3/64 

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
