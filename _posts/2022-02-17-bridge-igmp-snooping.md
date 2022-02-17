---
layout: post
title:  "Linux Networking Bridge and IGMP Snooping"
author: Joachim Wiberg
date:   2022-02-17 14:32:24 +0100
tags:
  - howto
  - bridge
---

This is the third post in a series of blog posts showing how to set up
networking in Linux using low-level tools.

In this part we talk about limiting the broadcast effects of multicast
using IGMP/MLD snooping in the Linux bridge (software switch).  Our
context, as usual, is industrial networking with a focus on embedded
devices.

<!-- more -->

## Introduction

The Linux bridge recently gained support for per-VLAN IGMP/MLD snooping.
It stands fine on its own legs, with lots of new per-VLAN settings.  But
since it lives in the bridge, it cannot know about any VLAN interfaces
that may (or may not) sit on top of the bridge.  There are naming issues
(br0.1 vs vlan1), and the fact that an interface can have multiple IP
addresses assigned to it.  Therefore, the current bridge implementation
in `br_multicast.c` can only[^1] act as a proxy querier, i.e., for IGMP
this means it can only send queries (per-VLAN) with source IP 0.0.0.0.

To understand why this might be a problem there are two things to
consider:

  1. Some networks don't have a (dynamic) multicast router[^2].  Usually
     the multicast router is the IGMP/MLD querier for a LAN, but some
     LANs consist only of (industrial) switches that try their best to
     limit the spread of multicast[^2] on low-capacity links or to end
	 devices sensitive to too much data
  2. Some end-devices discard queries with source IP 0.0.0.0.  This is
     of course wrong, but good luck telling a PLC vendor they should
     change their embedded firmware of an aging product, or even the
     customer site to upgrade their locked-down system with a brand new
     firmware -- these people don't like change and sometimes because it
     comes with drawn-out re-certification processes

So, proxy queries are allowed per RFC, but that might not work with some
end-devices, and we can't disable proxy-query because some end-devices
don't do gratuitous join/leave.  Hence, we need a service to provide us
with an IGMP/MLD querier that can (at least) use one of the IP address
on the interfaces we have on top of the bridge.  Here we present one
solution to that problem, called [querierd][].

However, first we need to talk a bit about IGMP and MLD, which are
foreign concepts to many, and magical buggy fairy dust that doesn't
work, to some.

> Actually, it's not at all uncommon that many IGMP/IGMP snooping
> implementations in the wild are buggy.  So you can find many (!) live
> networks out there that simply have disabled it on all switches just
> to get anything to work.  This is of course both unsafe and can cause
> a lot of overload on end devices, since unregulated multicast is then
> treated as broadcast.

[^1]: Well not really, there *is* support for using the IP address of
    the bridge, but you do not want to use that since the kernel will go
    dumpster diving for *any* address.
[^2]: Remember, unregulated multicast is broadcast.  Meaning you can
    easily run into overloading your networks.


## IGMP/MLD

IGMP/MLD snooping means a switch, or in this case the Linux bridge, can
"snoop" on the IGMP traffic from the elected querier and all multicast
receivers on a LAN.  [IGMP][] is the control protocol for IPv4 multicast
and [MLD][] is the control protocol for IPv6 multicast.  Both regulate
the flow of multicast on a LAN and are *very* similar, except for the IP
protocol.  We will focus on IPv4 in this post.

The basic mechanism in IGMP is similar to a newspaper subscription, a
sales person (querier) calls up potential customers (all end devices) in
their distribution area (LAN); "Do you want multicast? (Query)" and the
customers (end devices) answer (IGMP report) with the newspaper(s)
(group(s) they want to subscribe to.

As you can see, there are two types of IGMP/MLD control packets:

  1. IGMP/MLD query, sent by the elected querier
  2. IGMP/MLD report, also called "join" or "leave" from earlier
     protocol versions.  E.g., an end device wants to "join a group",
	 meaning; stop filtering this group for me

The IGMP/MLD *snooping mechanism* is responsible for:

  1. Forwarding queries from the elected querier to all end devices on
     the same LAN (members of the same VLAN).
  2. Forwarding responses to queries (reports) from end devices to the
     querier.

In both cases it then uses the information of which port the querier is
connected to, and which port(s) an IGMP/MLD reply was received on.
These ports are then used to selectively: flood multicast data towards
the querier, and forward multicast data (or not) to end devices.

   1. The port where queries are received, this is where all multicast
      data should be flooded -- it's the distribution point for all
      multicast, which may or may not be routed further.
   2. The port(s) which send multicast subscription replies (reports)
      are recorded in "filters", one filter per group (per VLAN) with a
      list of ports to forward to.

The bridge can be configured to, by default, flood or *not* flood
unknown multicast traffic to end devices.  In many setups you may want
to prohibit flooding by default, so that multicast is only forwarded if
an end device has subscribed to it.  However, due to limitations in the
underlying switching chipsets, this "block by default" approach may not
always be possible.  So, a compromise, which is also the Linux bridge's
default, is to have flooding of unknown/unregistered multicast per port
enabled by default, and start filtering when it knows more.  I.e., when
it has snooped its first IGMP/MLD report.


## Refresh

Recall the picture from the [Linux Networking Bridge][] post, we will
use a similar setup here, so if you want a refresh of the basics, see
that post.

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

There's a lot happening in that picture, so we'll start with a slightly
smaller example.


## Example

In this smaller example we have two VLANs, with end devices connected as
untagged members in both VLAN 1 and 2, in factm only the `br0` "port" is
a tagged member so we can set up our VLAN interfaces on top, which we
name `vlan1` and `vlan2`.  See the previously mentioned blog post for
how to set this up.

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

We start by enabling multicast snooping and IGMPv3 (legacy default is
IGMPv2) proxy querier on the bridge.  This is the load bearing feature
that we later build on:

    $ bridge vlan global set vid 1 dev br0 mcast_snooping 1 mcast_querier 1 mcast_igmp_version 3
    $ bridge vlan global set vid 2 dev br0 mcast_snooping 1 mcast_querier 1 mcast_igmp_version 3

> Enabling IGMP/MLD snooping per VLAN, and other per VLAN settings, on
> the bridge is only possible using iproute2 v5.16.0, or later.  In our
> examples we use [NetBox][], which has all the necessary tools and a
> matching kernel.  See that project for how to build the OS profile of
> the *Zero* target (x86_64) to play with this in Qemu (`make run`).
> Remember to set `QEMU_N_NICS=4`!

As soon as the vlan1 and vlan2 interfaces are UP, the bridge initiates
IGMP queries on both VLANs, so you should see unique queries going out
on all eth-ports.  In both [tcpdump(8)][] and Wireshark you can see the
IGMPv3 flag being set:

```
joachim@wbg:~$ tcpdump -plni qtap3 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on qtap3, link-type EN10MB (Ethernet), capture size 262144 bytes
17:26:05.697222 IP6 fe80::5054:ff:fe12:3456 > ff02::1: HBH ICMP6, multicast listener query v2 [gaddr ::], length 28
17:26:05.697345 IP 0.0.0.0 > 224.0.0.1: igmp query v3
```

To see all the per-VLAN settings:

```
root@zero:~# bridge vlan global show
port              vlan-id
br0               1-2
                    mcast_snooping 1 mcast_querier 1 mcast_igmp_version 3 mcast_mld_version 2 mcast_last_member_count 2 mcast_last_member_interval 100 mcast_startup_query_count 2 mcast_startup_query_interval 3125 mcast_membership_interval 26000 mcast_querier_interval 25500 mcast_query_interval 12500 mcast_query_response_interval 1000 
```

The query interval (125 sec) and router timeout (255 sec) are default
values from the RFCs.  Notice the scale of these, so if you want to
change them you don't set them too low!  See the RFC for recommended
defaults, e.g. timeout = 2 * 125 + 10/2.


## querierd

Now, to address the inherent problems of only having a proxy querier,
mentioned in the introduction.  Westermo developed a small and basic
querier daemon for userspace, called [querierd][].  It is derived from
the upstream [mrouted][] project and then whacked at with a bat to get
into shape for its new task.

querierd is installed by default in NetBox, it starts up by default on
`vlan1`, so all we have to do is uncomment the second to last line to
activate it also on our `vlan2` interface:

```conf
# /etc/querierd.conf: default NetBox configuration

# Query interval can be [1,1024], default 125.  Recommended not go below 10
#query-interval 125

# The interval inside the query-interval that clients should respond
#query-response-interval 10

# Last member query interval [1,1024], default 1.  The igmp-robustness
# setting controls the last member query count.
#query-last-member-interval 1

# Querier's robustness can be [2,10], default 2.  Recommended to use 2
#robustness 2

# Controls the "other querier present interval", used to detect when an
# elected querier stops sending queries.  Leave this commented-out, it
# is only available to override particular use-cases and interip with
# devices that do not follow the RFC.  When commented out, the timeout
# is calculated from the query interval and robustness according to RFC.
#router-timeout 255

# IP Option Router Alert is enabled by default, for interop with stacks
# that hard-code the length of the IP header
#no router-alert

# Enable and one of the IGMP versions to use at startup, with fallback
# to older versions if older clients appear.
iface vlan1 enable igmpv3
iface vlan2 enable igmpv3
#iface vlan3 enable igmpv3
```

If you want, you can uncomment also `vlan3`, even though it doesn't
exist (yet), querierd does not go to the sad place, it's just prepared
to automatically start as soon as it's added, UP, and has an IP address.

Remember to restart the daemon after you change its .conf file:

```
root@zero:~# initctl restart querierd
```

To see the status, we use the `querierctl` tool:

```
root@zero:~# querierctl -p
Multicast Overview
-------------------------------------------------------------------------------
Query Interval          : 125 sec
Robustness Value        : 2
Router Timeout          : 255
Fast Leave Ports        : 
Router Ports            : 
Flood Ports             : eth0, eth3, eth1, eth2

Interface         State     Querier               Timeout  Ver
-------------------------------------------------------------------------------
vlan1             Up        192.168.1.1           None       3
vlan2             Down      0.0.0.0               None       3

 VID  Multicast MAC         Multicast Group       Ports
-------------------------------------------------------------------------------
   1  01:00:5E:7F:FF:FA     239.255.255.250       br0, eth0
```

This reminds us that we haven't yet set an IP address, or even brought
the interface UP!  So we do that, and assign the proper address (without
restarting querierd):

```
root@zero:~# ip link set vlan2 up
root@zero:~# ip addr add 192.168.2.1/24 dev vlan2
root@zero:~# querierctl -p
Multicast Overview
-------------------------------------------------------------------------------
Query Interval          : 125 sec
Robustness Value        : 2
Router Timeout          : 255
Fast Leave Ports        : 
Router Ports            : 
Flood Ports             : eth0, eth3, eth1, eth2

Interface         State     Querier               Timeout  Ver
-------------------------------------------------------------------------------
vlan1             Up        192.168.1.1           None       3
vlan2             Up        192.168.2.1           None       3

 VID  Multicast MAC         Multicast Group       Ports
-------------------------------------------------------------------------------
   1  01:00:5E:7F:FF:FA     239.255.255.250       br0, eth0
```

## Exercise

Now, as an exercise to the reader, you can start playing around with the
setup.  Emulate IGMP reports from end devices, investigate the various
querierctl commands, and compatibility output options.  E.g., we use
`-p` here only to make the output more blog friendly.

Also, try connecting multiple Zero devices, using host bridging, test
querier election (lowest IP in a LAN wins, except for 0.0.0.0), and
router (querier) timeout with fail-over to another querierd.


## Future

Much if the original innards of querierd have been gutted and new DNA
strands grafted onto it from other related projects, such as [pimd][].
The only missing part, currently, is the MLD v1/v2 querier support,
which is planned to be grafted from [pim6sd][].

There are however other possibilities, one such is to convert it to a
full-blown `bridged` (plans for this are currently shelved), which can
take care of all the layer-2 bridge setup (port VLAN memberships, and
all per-VLAN settings) and active functions like a querier for IGMP/MLD
but also GMRP/MMRP for MAC based multicast subscription, and other types
of layer-2 services and policy that don't belong in the kernel.


[tcpdump(8)]: https://www.man7.org/linux/man-pages/man8/tcpdump.8.html
[Linux Networking Bridge]: /2020/03/25/linux-networking-bridge/
[IGMP]:       https://datatracker.ietf.org/doc/html/rfc3376
[MLD]:        https://datatracker.ietf.org/doc/html/rfc3810
[querierd]:   https://github.com/westermo/querierd/
[NetBox]:     https://github.com/westermo/netbox/
[mrouted]:    https://github.com/troglobit/mrouted/
[pimd]:       https://github.com/troglobit/pimd/
[pim6sd]:     https://github.com/troglobit/pim6sd/
