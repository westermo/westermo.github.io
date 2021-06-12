---
layout: post
title:  "Running Foreign Containers on LXD"
author: Tobias Waldekranz
date:   2020-03-02 15:43:00 +0100
categories: containers
---

<!-- more -->

Recently, we added LXD support to [myrootfs][], to make it easy to test
a container on your local machine before deploying it to the target
device.  In addition to wrapping the basics of importing images and
launching containers, there is also support for handling containers of
foreign architectures.  This is very useful in embedded scenarios, as it
allows you to test your container on your development system before
deploying to the target.

All of this has been integrated to [myrootfs][] so that you don't have
worry about it. This post is a behind-the-scenes look at how LXD can be
configured to make it work.


Native Container
----------------

We'll start easy, creating a native container (i.e. `x86_64` in my
case):

```
$ make ARCH=x86 defconfig && make
```

Once this is done we'll have a SquashFS image available in
`images/`. In order to import it to LXD, we create a `metadata.yaml`
file to describe it:

```yaml
architecture: x86_64
creation_date: 1582833008
properties:
  os: myrootfs
  release: 1.0.0
  description: myrootfs-1.0.0
```

LXD expects the metadata in a tarball, to we'll put it in one:

```
$ tar caf metadata.tar.gz metadata.yaml
```

Then we can import it to LXD:

```
$ lxc image import --alias myrootfs metadata.tar.gz myrootfs.img
```

Now we can create a container and attach to it:

```
$ lxc launch -e myrootfs
Creating the instance
Instance name is: firm-kite
Starting firm-kite
~/src/github.com/myrootfs/myrootfs/images(master)$ lxc console firm-kite
To detach from the console, press: <ctrl>+a q

/ # uname -a
Linux myrootfs 5.3.0-29-generic #31-Ubuntu SMP Fri Jan 17 17:27:26 UTC 2020 aarch64 GNU/Linux
/ #
$ lxc stop firm-kite
```

Well, that was pretty easy. I think we're ready to tackle a
cross-compiled container now!


Foreign Container
-----------------

First, let's install the statically linked version of QEMU's usermode
emulation binaries. On Debian derivatives, this is available in the
package `qemu-user-static`:

```
# apt install qemu-user-static
```

Importantly, this will also take care of setting up `binfmt_misc`
entries for all supported architectures.

We're now ready to build our foreign container. We could pick any
architecture supported by [myrootfs]. I asked my magic 8-ball and it
told me to use `aarch64`, and who am I to tempt the gods.

```
$ make ARCH=arm64 generic_defconfig && make
```

LXD will refuse to even attempt to start a container of a different
architecture than the host's. Fortunately we can just lie to it by
using the same metadata file as in the previous section.

We can then import it just like before:

```
$ lxc image import --alias myrootfs-aarch64 metadata.tar.gz myrootfs.img
```

As expected, launching the container will not end well:

```
$ lxc launch myrootfs-aarch64
Creating the container
Container name is: above-boar
Starting above-boar
Error: Failed to run: /usr/lib/lxd/lxd forkstart above-boar /var/lib/lxd/containers /var/log/lxd/above-boar/lxc.conf:
Try `lxc info --show-log local:above-boar` for more info
$ lxc info --show-log local:above-boar
Name: above-boar
Remote: unix://
Architecture: x86_64
Created: 2020/03/02 08:15 UTC
Status: Stopped
Type: persistent
Profiles: default

Log:

lxc above-boar 20200302081554.516 WARN     conf - conf.c:lxc_setup_devpts:1616 - Invalid argument - Failed to unmount old devpts instance
lxc above-boar 20200302081554.519 ERROR    start - start.c:start:2028 - No such file or directory - Failed to exec "/sbin/init"
lxc above-boar 20200302081554.519 ERROR    sync - sync.c:__sync_wait:62 - An error occurred in another process (expected sequence number 7)
lxc above-boar 20200302081554.519 WARN     network - network.c:lxc_delete_network_priv:2589 - Operation not permitted - Failed to remove interface "eth0" with index 235
lxc above-boar 20200302081554.519 ERROR    lxccontainer - lxccontainer.c:wait_on_daemonized_start:842 - Received container state "ABORTING" instead of "RUNNING"
lxc above-boar 20200302081554.520 ERROR    start - start.c:__lxc_start:1939 - Failed to spawn container "above-boar"
lxc 20200302081554.537 WARN     commands - commands.c:lxc_cmd_rsp_recv:132 - Connection reset by peer - Failed to receive response for command "get_state"
```

The kernel is configured to run `aarch64` binaries using the
interpreter `/usr/bin/qemu-aarch64-static` using `binfmt_misc`:

```
$ cat /proc/sys/fs/binfmt_misc/qemu-aarch64
enabled
interpreter /usr/bin/qemu-aarch64-static
flags: OC
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
```

But this binary is not available inside the LXD container. We can
change that by adding a "device" in LXD parlance which bind mounts in
the binary we need. This can be done in different ways, I will create
a profile and apply it to our container, that makes it easy to reuse
in the future. The profile looks like this:

```yaml
devices:
  qemu-aarch64-static:
    path: /usr/bin/qemu-aarch64-static
    source: /usr/bin/qemu-aarch64-static
    type: disk
```

Why is this a "device", and of type "disk" nonetheless, you might
ask. Well I don't know what to tell you, that's the names they
chose. Points for originality I guess.

A profile is created and applied to the instance like so:

```
$ lxc profile create qemu-aarch64
Profile qemu-aarch64 created
$ lxc profile edit qemu-aarch64 <<EOF
> devices:
>   qemu-aarch64-static:
>     path: /usr/bin/qemu-aarch64-static
>     source: /usr/bin/qemu-aarch64-static
>     type: disk
> EOF
$ lxc profile add above-boar qemu-aarch64
Profile qemu-aarch64 added to above-boar
```

If we start it again:

```
$ lxc start above-boar
$ lxc console above-boar
To detach from the console, press: <ctrl>+a q

/ # uname -a
Linux myrootfs 4.15.0-76-generic #86-Ubuntu SMP Fri Jan 17 17:24:28 UTC 2020 aarch64 GNU/Linux
/ # ps
  PID USER       VSZ STAT COMMAND
    1 root     57760 S    {init} /usr/bin/qemu-aarch64-static /sbin/init
  166 root     57760 S    {udhcpc} /usr/bin/qemu-aarch64-static /sbin/udhcpc -R -n -p /var/run/udhcpc.et
  175 root     57760 S    {syslogd} /usr/bin/qemu-aarch64-static /sbin/syslogd -b 3 -S -D -L
  180 root     56728 S    {dropbear} /usr/bin/qemu-aarch64-static /sbin/dropbear -R -B -K 30 -I 900
  182 root     57760 S    {sh} /usr/bin/qemu-aarch64-static /bin/sh
  185 root         0 0W   {/bin/ps} /bin/ps
/ #
```

Success!


[myrootfs]: https://github.com/myrootfs/myrootfs
