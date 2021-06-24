---
layout: post
title:  "Bridge Forwarding Problem"
author: Joachim Wiberg
date:   2021-06-24 10:21:42 +0100
tag: troubleshooting
---

Recent Linux distributions (2021) have enabled Bridge Firewalling by
default.  In particular on Ubuntu 21.04 this has been know to cause
a fair amount of head scratching!

This blog post shows how to rectify the problem using `sysctl`.

<!-- more -->

The relevant bridge settings can be seen with `sysctl`:

```
$ sysctl net.bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-filter-pppoe-tagged = 0
net.bridge.bridge-nf-filter-vlan-tagged = 0
net.bridge.bridge-nf-pass-vlan-input-dev = 0
```

The culprits are the first three enabled bridge netfilter settings.
They enable hooks in the bridge for trapping and dropping frames on
different layers.

We create the file `/etc/sysctl.d/90-bridge-no-filter.confg`
by using a clever HERE script:

```
$ sudo tee /etc/sysctl.d/90-bridge-no-filter.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables  = 0
net.bridge.bridge-nf-call-arptables = 0
EOF
```

Activate the new settings by restarting the systemd service, or
rebooting your system:

```
$ sudo systemctl restart systemd-sysctl.service
```

Verify that the new settings took:

```
$ sysctl net.bridge
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
...
```

For more information on this topic, see the following libVirt wiki page:  
🖙 <https://wiki.libvirt.org/page/Net.bridge.bridge-nf-call_and_sysctl.conf>

