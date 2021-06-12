---
layout: post
title:  "Hairpin Multicast Routing"
author: Tobias Waldekranz
date:   2020-01-17 09:36:42 +0100
tag: bugstory
---

<!-- more -->

Problem
=======

For a given IP multicast group, if the following conditions are true:

- There is a "hairpin" multicast route, i.e. the ingress interface is
  equal to the egress interface, configured for that group.
- There is an open local socket, which has joined (in IGMP terms) that
  same group.

Linux will send out `TTL - 1` copies to the network. Instead of the
expected: `1`.


Reproduction
============

On the target machine, use `smcroute` to setup a hairpin route and
perform a join on that same group:

```
# smcrouted
# smcroutectl add  eth0 198.18.1.1 239.1.1.1 eth0
# smcroutectl join eth0            239.1.1.1
```

Inject traffic to that group with this `trafgen` payload:
```
{
        eth(sa=02:00:00:00:00:01, da=01:00:5e:01:01:01),
        ipv4(saddr=198.18.1.1, daddr=239.1.1.1, ttl=5)
        udp(sp=1, dp=2),
        "Hello world"
}
```

Snooping `eth0` on the target:
```
# tcpdump -vnli eth0 multicast
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
15:07:28.395793 IP (tos 0x0, ttl 5, id 0, offset 0, flags [none], proto UDP (17), length 39)
    198.18.1.1.1 > 239.1.1.1.2: UDP, length 11
15:07:28.395852 IP (tos 0x0, ttl 4, id 0, offset 0, flags [none], proto UDP (17), length 39)
    198.18.1.1.1 > 239.1.1.1.2: UDP, length 11
15:07:28.395898 IP (tos 0x0, ttl 3, id 0, offset 0, flags [none], proto UDP (17), length 39)
    198.18.1.1.1 > 239.1.1.1.2: UDP, length 11
15:07:28.395906 IP (tos 0x0, ttl 2, id 0, offset 0, flags [none], proto UDP (17), length 39)
    198.18.1.1.1 > 239.1.1.1.2: UDP, length 11
15:07:28.395912 IP (tos 0x0, ttl 1, id 0, offset 0, flags [none], proto UDP (17), length 39)
    198.18.1.1.1 > 239.1.1.1.2: UDP, length 11
^C
5 packets captured
5 packets received by filter
0 packets dropped by kernel
#
```
We can see the incoming frame, with `TTL=5` being routed back four
times, decrementing the TTL by one for each packet.


Investigation
=============

How are we getting here? Make stacks!
```
# ply 'k:dev_queue_xmit { print(stack); }'
info: creating kallsyms cache
ply: active

	dev_queue_xmit+1
	ip_finish_output+318
	ip_mc_output+139
	NF_HOOK.constprop.52+228
	ipmr_queue_xmit.isra.39+982
	ip_mr_forward+325
	ip_mr_input+323
	ip_rcv_finish+125
	ip_rcv+217
	__netif_receive_skb_one_core+80
	__netif_receive_skb+19
	netif_receive_skb_internal+57
	napi_gro_receive+216
	receive_buf+1188
	virtnet_poll+230
	net_rx_action+619
	__softirqentry_text_start+235
	irq_exit+181
	do_IRQ+86
	common_interrupt+15
	native_safe_halt+18
	__cpuidle_text_start+24
	arch_cpu_idle+10
	default_idle_call+30
	do_idle+450
	cpu_startup_entry+110
	rest_init+188
	start_kernel+1207
	x86_64_start_reservations+42
	x86_64_start_kernel+114
	secondary_startup_64+164


	dev_queue_xmit+1
	ip_finish_output+318
	ip_mc_output+139
	NF_HOOK.constprop.52+228
	ipmr_queue_xmit.isra.39+982
	ip_mr_forward+325
	ip_mr_input+323
	ip_rcv_finish+125
	ip_rcv+217
	__netif_receive_skb_one_core+80
	__netif_receive_skb+19
	process_backlog+191
	net_rx_action+619
	__softirqentry_text_start+235
	run_ksoftirqd+50
	smpboot_thread_fn+357
	kthread+253
	ret_from_fork+31


	dev_queue_xmit+1
	ip_finish_output+318
	ip_mc_output+139
	NF_HOOK.constprop.52+228
	ipmr_queue_xmit.isra.39+982
	ip_mr_forward+325
	ip_mr_input+323
	ip_rcv_finish+125
	ip_rcv+217
	__netif_receive_skb_one_core+80
	__netif_receive_skb+19
	process_backlog+191
	net_rx_action+619
	__softirqentry_text_start+235
	run_ksoftirqd+50
	smpboot_thread_fn+357
	kthread+253
	ret_from_fork+31


	dev_queue_xmit+1
	ip_finish_output+318
	ip_mc_output+139
	NF_HOOK.constprop.52+228
	ipmr_queue_xmit.isra.39+982
	ip_mr_forward+325
	ip_mr_input+323
	ip_rcv_finish+125
	ip_rcv+217
	__netif_receive_skb_one_core+80
	__netif_receive_skb+19
	process_backlog+191
	net_rx_action+619
	__softirqentry_text_start+235
	run_ksoftirqd+50
	smpboot_thread_fn+357
	kthread+253
	ret_from_fork+31

^Cply: deactivating
#
```

Ok, first packet is going out as expected, the other four are coming
from the `process_backlog` path, meaning that we're most likely
calling `enqueue_to_backlog` somewhere:

```
# ply 'k:enqueue_to_backlog { print(stack); exit(0); }'
ply: active

	enqueue_to_backlog+1
	netif_rx_ni+33
	dev_loopback_xmit+154
	ip_mc_finish_output+39
	ip_mc_output+352
	NF_HOOK.constprop.52+228
	ipmr_queue_xmit.isra.39+982
	ip_mr_forward+325
	ip_mr_input+323
	ip_rcv_finish+125
	ip_rcv+217
	__netif_receive_skb_one_core+80
	__netif_receive_skb+19
	netif_receive_skb_internal+57
	napi_gro_receive+216
	receive_buf+1188
	virtnet_poll+230
	net_rx_action+619
	__softirqentry_text_start+235
	irq_exit+181
	do_IRQ+86
	common_interrupt+15
	native_safe_halt+18
	__cpuidle_text_start+24
	arch_cpu_idle+10
	default_idle_call+30
	do_idle+450
	cpu_startup_entry+110
	rest_init+188
	start_kernel+1207
	x86_64_start_reservations+42
	x86_64_start_kernel+114

ply: deactivating
#
```

Looking at the code:

```c
	if (rt->rt_flags&RTCF_MULTICAST) {
		if (sk_mc_loop(sk)
#ifdef CONFIG_IP_MROUTE
		/* Small optimization: do not loopback not local frames,
		   which returned after forwarding; they will be  dropped
		   by ip_mr_input in any case.
		   Note, that local frames are looped back to be delivered
		   to local recipients.

		   This check is duplicated in ip_mr_input at the moment.
		 */
		    &&
		    ((rt->rt_flags & RTCF_LOCAL) ||
		     !(IPCB(skb)->flags & IPSKB_FORWARDED))
#endif
		   ) {
			struct sk_buff *newskb = skb_clone(skb, GFP_ATOMIC);
			if (newskb)
				NF_HOOK(NFPROTO_IPV4, NF_INET_POST_ROUTING,
					net, sk, newskb, NULL, newskb->dev,
					ip_mc_finish_output);
		}

		/* Multicasts with ttl 0 must not go beyond the host */

		if (ip_hdr(skb)->ttl == 0) {
			kfree_skb(skb);
			return 0;
		}
	}
```

We see that the reason that we're not getting any replicated packets
unless we join the group is that `rt->rt_flags & RTCF_LOCAL` will be
false in that case. However, the other guard should still apply since
the packet has been routed. The comment also references a duplicated
check in `ip_mr_input`:
```c
	/* Packet is looped back after forward, it should not be
	 * forwarded second time, but still can be delivered locally.
	 */
	if (IPCB(skb)->flags & IPSKB_FORWARDED)
		goto dont_forward;
```

Alright, so everything should be great then! Alas, it is not. We are
sad. This must mean that the flag is cleared somewhere before the
duplicated packet reaches `ip_mr_input`.

Using GDB, we can verify that the flag is indeed set when we enqueue
the packet:

```
Temporary breakpoint 11, enqueue_to_backlog (skb=0xffff88800aa84200, cpu=0, qtail=0xffff88800de03b00) at net/core/dev.c:4225
4225	{
(gdb) print ((struct inet_skb_parm*)((skb)->cb))->flags
$7 = 1
```

Still set when we come back to process the queue:
```
Temporary breakpoint 13, process_backlog (napi=0xffff88800de222d0, quota=64) at net/core/dev.c:5855
5855				__netif_receive_skb(skb);
(gdb) p skb
$9 = (struct sk_buff *) 0xffff88800aa84200
(gdb) print ((struct inet_skb_parm*)((skb)->cb))->flags
$10 = 1
```

And when entering the IP stack:
```
Temporary breakpoint 14, ip_rcv (skb=0xffff88800aa84200, dev=0xffff88800f58a000, pt=0xffffffff82308ce0 <ip_packet_type>, orig_dev=0xffff88800f58a000) at net/ipv4/ip_input.c:518
518	{
(gdb) print ((struct inet_skb_parm*)((skb)->cb))->flags
$11 = 1
```

But not after pre-routing:
```
Temporary breakpoint 15, ip_rcv_finish (net=0xffffffff822a9700 <init_net>, sk=0x0 <irq_stack_union>, skb=0xffff88800aa84200) at net/ipv4/ip_input.c:401
401	{
(gdb) print ((struct inet_skb_parm*)((skb)->cb))->flags
$12 = 0
```

**Aside:** Yes, I am aware of watchpoints, but I could not get them to
work for some reason. When in doubt, brute force!

This means that we're dropping the flag in `ip_rcv`, which is mostly
just a wrapper for `ip_rcv_core` in which we find this:

```c
	/* Remove any debris in the socket control block */
	memset(IPCB(skb), 0, sizeof(struct inet_skb_parm));
```

Thus, when we reach `ip_mr_input` the `IPSKB_FORWARDED` will __NEVER__
be set. As a result, the packet will keep being replicated until the
TTL runs out.


Discussion
==========

Why is this code here:

```c
	/* Remove any debris in the socket control block */
	memset(IPCB(skb), 0, sizeof(struct inet_skb_parm));
```

It is because this routine has no way of assuming that the contents of
`skb->cb` (which is the backing storage for `IPCB(skb)` was previously
set by the IP stack. It could just as well have been used by
underlying interface code (e.g. a bridge).

In order for this to work as expected, the flag needs to be saved in a
dedicated space on the `skb` that is not touched by other layers.

So why even perform the check then? Well, looking at the `git blame`,
we can see that the code is coming from the initial GIT commit by
Torvalds. It is very possible that this check has just been copied
from the BSD from which it came, where perhaps the information was
still available. That's a rabbit hole for another day though.


Solution
========

At a high level, we need some way for the IPMR stack to know that
we've passed through it before, and already performed the
routing. There are a few ways that I can think of to accomplish this:

1. Add a field to `struct sk_buff`.
  - Pros: Simple.
  - Cons: New `skb`-fields are seldom appreciated upstream, for good
    reason.
2. Use the new `skb_ext_*` functionality to append a TLV fragment to
  the `skb`.
  - Pros: Only impacts the `skb` in this particular corner-case.
  - Cons: A few more lines of code.

IMHO, (2) looks like the clear winner. In `ip_mc_finish_output` we
could `skb_ext_add` an `SKB_EXT_IPMR_LOOPBACK` extension indicating
that the packet was forwarded. Something like:

```patch
@@ -328,6 +328,9 @@ static int ip_mc_finish_output(struct net *net, struct sock *sk,
 		return ret;
 	}
 
+	if (IPCB(skb)->flags & IPSKB_FORWARDED)
+		skb_ext_add(skb, SKB_EXT_IPMR_LOOPBACK);
+
 	return dev_loopback_xmit(net, sk, skb);
 }
```

This could then be retrieved in `ip_mr_input`:

```patch
@@ -2103,6 +2103,11 @@ int ip_mr_input(struct sk_buff *skb)
 		}
 	}
 
+	if (unlikely(skb_ext_exist(skb, SKB_EXT_IPMR_LOOPBACK))) {
+		IPCB(skb)->flags |= IPSKB_FORWARDED;
+		skb_ext_put(skb, SKB_EXT_IPMR_LOOPBACK);
+	}
+
 	/* Packet is looped back after forward, it should not be
 	 * forwarded second time, but still can be delivered locally.
 	 */
```
