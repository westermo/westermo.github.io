---
layout: post
title:  "Tools of the Trade :: Debugging an embedded system"
author: Joachim Wiberg
date:   2022-02-18 15:42:24 +0100
tags:
  - howto
  - tools
  - gdb
  - gdbserver
  - buildroot
---

This post shows how to debug an embedded system with gdbserver.  We will
run the [Buildroot][] derivative [NetBox][] in Qemu to debug a program
that segfaults.

> Throughout this blog post, the nomenclature for command prompts is:  
>   `# foo` run command `foo` as root on the target system  
>   `$ bar` run command `bar` as your own user on the host system

<!-- more -->

NetBox comes with native support for running all supported targets in
Qemu, including support for remote debugging.  The support includes a
lot of boilerplate to simplify connection and set up.  In this blog post
we will start by showing the bare essentials needed and then show the
simplified approach offered by NetBox.

> See the documentation for the [NetBox][] project on how to build the
> OS profile of the *Zero* target (x86_64) to play with it in Qemu.
> **Remember** to activate "Copy gdb server to the target" in the
> "Toolchain" menu, and "build packages with debugging symbols" in the
> "Build options" menu.

## Qemu

Thanks to the insanely powerful laptops developers have today, it's
possible to come very close to the actual HW performance when emulating
it in Qemu.  This unlocks many interesting advantages, like testing and
debugging, before deploying to the target device.

Qemu in NetBox is set up such that it enables debugging both of kernel
and user space.  The former is not covered in this post, we will only
scratch the surface of the latter.

Issuing `make run QEMU_GDB=1` in NetBox, opens two local TCP ports where
you can connect your gdb to.  For kernel debugging it's the completely
[random port number][really] 4711, and for user space port 4712.  These
can be changed using environment variables.

For user space debugging we also need to create a dedicated debug
console, called `hvc1`.  The default console in NetBox is `hvc0`.
This is done by adding the following to the Qemu command line:

    addr="port=${QEMU_GDB_UPORT:-4712},host=localhost"
    qemu-system-foo ... -gdb tcp::${QEMU_GDB_KPORT:-4711}
                        -chardev socket,id=hvc1,${addr},server=on,wait=off
                        -device virtconsole,name=gdb,chardev=hvc1
                    ...

The `...` marks other arguments.  See the [qemu wrapper script][qemu]
for details.


## gdbserver

When the target system boots (in Qemu), we start `gdbserver` with the
argument `/dev/hvc1` which is what allows us to connect to the target.
This is handled automatically behind the scenes, all you need to do as
a developer is to enable the service:

    # initctl enable gdbserver
    # initctl reload

The (default) command arguments this evaluates to are:

    # gdbserver --multi /dev/hvc1

To connect to actual HW devices you need to change the arguments to
instead open an actual TCP port.  See the file `/etc/default/gdbserver`
to change this -- after editing the file, do `initctl reload` again to
tell Finit to activate the change.

> **Tip:** you can of course also run `gdbserver` standalone from the
> command line.  Having it run as s service, though, allows you to
> attach to any process on the system, which is hugely useful when
> developing a system.


## Basic GDB Setup

So, now we're almost there, ready to fire up gdb and connect to the
target system!  Just one small hurdle left to clear, you need the
*correct* gdb installed:

    $ sudo apt install gdb-multiarch

Because, if you've come this far in the blog post, your target system is
not an x86_64 (or Apple M1), it's an ARM, MIPS, PowerPC, or Risc V.

In a Buildroot based system the unstripped binaries are installed in the
`output/staging/` directory by default.  This is a good time as any to
verify that you've enabled `BR2_ENABLE_DEBUG`, see "Build options".

Without the tight integration of gdb in NetBox, what you need is to do
the following in gdb to connect to the target, and attach to a running
process to debug.

    $ cd output/staging/
    $ gdb-multiarch
    (gdb) target extended-remote localhost:4712
    (gdb) set solib-absolute-prefix .
    (gdb) set solib-search-path lib:usr/lib
    (gdb) set remote exec-file /usr/sbin/myprogram
    (gdb) file usr/sbin/myprogram
    (gdb) attach 1234
    (gdb) cont

Where `1234` is the PID of your already running program.  Since this is
quite tedious, you can save this to a `.gdbinit` file, which gdb loads
when starting up.  This is what we've done to simplify things in NetBox.


## Simplified GDB with NetBox 

NetBox leverages the `$O` variable in Buildroot, and this is core to the
top level `Makefile` of NetBox.  It knows what target you are building
for, so if you have `O=~/src/netbox/output-zero` this is respected for
all commands to make, e.g., typing `make run QEMU_GDB=1` starts Qemu for
Zero.  The same is true for:

    $ make debug

This basically does the same as shown above, all you need to do is to
issue the appropriate commands (NetBox extensions):

    For help, type "help".
    (gdb) user-connect
    (gdb) user-attach usr/sbin/querierd 488
    0x00007f5812afc425 in select () from ./lib64/libc.so.6
    (gdb) cont
    Continuing.


[NetBox]: https://github.com/westermo/netbox/
[qemu]:   https://github.com/westermo/netbox/blob/master/utils/qemu
[really]: https://www.urbandictionary.com/define.php?term=4711
