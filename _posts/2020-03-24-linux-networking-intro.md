---
layout: post
title:  "Linux Networking Introduction"
author: Joachim Wiberg
date:   2020-03-24 19:22:42 +0100
tag: howto
---

Every engineer and sysadmin working with Linux have at one time or
another run into networking problems.  This post is the first in a
series showing how to set up networking using low-level tools like:
`ip`, `bridge`, `tc`, and `ethtool`.

Let's start by exploring the basic concepts of networking.

<!-- more -->

## LAN and Broadcast Domains

A Local Area Network, or LAN, is a shared Ethernet or Wi-Fi network of
end devices, which allows unicast, multicast, and broadcast traffic.
Devices on a LAN today usually communicate using the Internet Protocol
(IP), and are commonly using IPv4, more on that below.

A brodcast domain is a LAN.  Many LAN segments can be bridged to create
a larger broadcast domain.  To facilitate communication between LANs you
need a router, sometimes referred to as gateway.  More on bridging and
routing later.

All devices have an IP address, but to actually communicate with a
neighbor on a LAN, a device must find the neighbors physical address,
the MAC address.

The process in IPv4 to find the MAC address for IP address 192.168.2.200
is to send a broadcast message to everyone on the LAN:

    ARP who has 192.168.2.200?

Everyone who isn't 192.168.2.200 will ignore and drop the message.  But the
device which has address 192.168.2.200 will respond:

    ARP 192.168.2.200 is at 00:07:7c:c0:ff:ee

ARP stands for Address Resolution Protocol.  When debugging low-level
networking problems, this is usually the first suspect to investigate.
Useful tools for debugging ARP related problems are:

- [arp(8)](http://man7.org/linux/man-pages/man8/arp.8.html)
- [ip-neighbour(8)](http://man7.org/linux/man-pages/man8/ip-neighbour.8.html)
- [arpping(8)](http://man7.org/linux/man-pages/man8/arping.8.html)

> Needless to say, ARP is crucial for IPv4 networking.  When in doubt,
> use [tcpdump](https://en.wikipedia.org/wiki/Tcpdump), `tshark`, or
> [wireshark](https://en.wikipedia.org/wiki/Wireshark) to debug.  On
> most small embedded targets only `tcpdump` exists, it can save `.pcap`
> format which can be more easily inspected with Wireshark.

The astute reader probably noticed that sending broadcast messages is
quite a waste, since most devices on the network will have to wake up,
read the message only to throw it away a nanosecond later.  In IPv6, ARP
has been replaced with another mechanism, not covered here.  In fact, in
IPv6 broadcast does not exist at all.

## IP Networking

> This HowTo series will focus entirely on IPv4 networking.

As explained in the previous section, IP Networking relies on Ethernet
or Wi-Fi networks where the physical, or MAC, address is required for
communication.

As a very short introduction to IPv4 networking we present a few basic
concepts:

- IPv4 address, e.g. 192.168.2.200
- IPv4 netmask, e.g. 255.255.255.0
- IPv4 gateway, e.g. 192.168.2.1

IP Networks work fine without a gateway/router.  The only requirement is
that all end devices share the same LAN (broadcast domain) and the same
IP network.

> Multiple IP networks may share the same LAN, called *multinetting*.

The address is unique to a single device.  It maps 1:1 to the physical
MAC address of a PC network interface card (NIC), or similar.  A NIC may
however have multiple IP addresses.

The netmask is common to all devices in the same IP subnet.  In this
example we've chosen a classic Class-C network, because they are
probably the easiest to wrap your brain around.  The netmask is
sometimes written in a `/LEN` syntax, in the case of 255.255.255.0 it
corresponds to /24, because there are 24 binary 1's, counted from the
left before the first binary 0.

Hence, to denote a network one can write one of:

- 192.168.2.0 netmask 255.255.255.0, or
- 192.168.2/24

This network has two reserved addresses; then network 192.168.2.0 and
the broadcast address 192.168.2.255.  All other addresses can be used
by end devices.

In comparison, a Class-B network could look like this:

- 172.16.0.0 netmask 255.255.0.0, or
- 172.16/16

This is a much bigger network, it can have 65534 end devices.  The
reserved network address is 172.16.0.0, but the broadcast address is
172.16.255.255.

> The whole range of IP addresses in 192.168.xxx.xxx are called *private*,
> as such they are reserved for LANs and must never be exposed on the
> Internet.  There exist more private IP ranges that can be used.

## Setting IP address and default route in Linux

Unless you are on a desktop Linux distribution, where much of the
networking magic is handled by high-level interfaces like
[NetworkManager](https://en.wikipedia.org/wiki/NetworkManager),
you need some way of setting IP address and default route.

There are two major approaches:

- static IP address assignment and default route
- dynamic assignment using DHCP

To set the IP address of an interface, here `eth0`, in Linux there are
both the traditional `ifconfig` tool and the new `ip address` command:

    ifconfig eth0 192.168.2.200 netmask 255.255.255.0
	
or

    ip address add 192.168.2.200/24 dev eth0

Inspect interface addresses using

    ifconfig
	
or

    ip address

Setting the default route (gateway) can be done using the traditional
`route` tool or the `ip route` command:

    route default gw 192.168.2.1
	
or

    ip route add default via 192.168.2.1
	
Most users are however happy just to get an IP address dynamically and
don't really care where it comes from.  However, for this to work you
need a DHCP server on the same LAN (or use a proxy, called DHCP relay,
but more about that in another post).  Just about all small office, home
and industrial gateways have built-in DHCP servers, most of them use the
amazing [dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq), more about
that in a later blog post.  There are many DHCP clients available, see
your particular system for details, here are a few:

- dhclient, https://www.isc.org/dhcp/
- pump, available originally in RedHat
- dhcpcd, https://roy.marples.name/projects/dhcpcd/
- udhcpc, available in [BusyBox](https://www.busybox.net) on most embedded systems

Setting up the DHCP client is usually not required on most desktop
systems, as it is the default.  In the GUI there's usually just a
simple checkbox or radio button.  Here are a few pointers though:

- https://wiki.debian.org/DHCP_Client
- https://wiki.archlinux.org/index.php/dhcpcd
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/deployment_guide/s1-dhcp-configuring-client


## Summary and More

In this post we've touched upon the topic of local area networks (LANs)
and broadcast domains.  Here we talked briefly about ARP and its role in
establishing communication between two devices on a LAN.

We then proceeded to the topic of IP networking and how to set an IP
address and default route in Linux.

But wait, even if we now understand how to communicate on a LAN, how can
we communicate between separate LANs?  Well ... remember the gateway?

The IP stack in your device knows how to talk to devices in the same IP
subnet (see above), for everything else it checks the routing table.  In
most end-devices there are two routes, in our case:

    # route -n
    Kernel IP routing table
    Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
    0.0.0.0         192.168.2.1     0.0.0.0         UG    0      0        0 eth0
    192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0

Our device has IP address 192.168.2.200, which falls in the net route
(last line), but for everything else it will use the default route.
I.e., if we want to talk to 192.168.1.200 in a neighboring network,
we send an ARP for 192.168.2.1.  Our gateway responds with its MAC
address, and we use that as the destination address of all frames,
but the IP header has the destination of 192.168.1.200.  When our
frames reach the gateway it can see, from the IP header, that it
is not the final destination, so it in turn checks *its* routing
table to see if it has a net route or must forward our frames to
yet another gateway.

## EOF

There is of course more, like; how does DNS work, and where can I learn
more about DHCP relay agents?  Don't worry, there will be more network
related posts coming up, and you can always contact Westermo for more
information, courses, and even some helpful products.

Visit <https://www.westermo.com>

