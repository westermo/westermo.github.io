---
layout: post
title:  "Tools of the Trade :: dnsmasq & conserver"
author: Joachim Wiberg
tags:
  - howto
  - tools
date:   2021-06-10 16:37:42 +0100
---

This is the first post, in hopefully a series, detailing tools we
networking and switch geeks love.  We start with the underrated
and oft forgotten dnsmasq and conserver.

  - [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) will be used
    to hand out BOOTP/DHCP leases to our embedded systems, as well as
    serve firmware images using TFTP

  - [conserver](https://www.conserver.com/) will be used to connect to
    our device's console port so we can access the bootloader and
    net-boot it from dnsmasq

<!-- more -->

## Install

Westermo is almost exclusively a Debian/Ubuntu and Linux Mint shop, so
all package install will be done using `apt`, the same packages exist in
other Linux distros as well.

    sudo apt install dnsmasq conserver-server conserver-client

Done.  We now proceed to configure the daemons, please note, this is
only an example and you can set them up in a myriad of ways.


## Configure dnsmasq

Some systems may have dnsmasq installed, either the system itself uses
it as a caching DNS (not mentioned in this blog post), or you have
libVirt and virt-manager installed.  That is fine, this blog post allows
you to have multiple configurations running at the same time, yet still
be quite user friendly.

The traditional `/etc/dnsmasq.conf` has today mostly been replaced with
the new `/etc/dnsmasq.d/` directory where you can add multiple files.

Let's create `/etc/dnsmasq.d/foo.conf`, please note the interface names
are for my PC, yours may be very different:

```
# You may need to uncomment this, depending on other
# services also using dnsmasq
#bind-dynamic
bootp-dynamic

dhcp-host=id:basis,set:basis
dhcp-host=id:coronet,set:coronet

enable-tftp
tftp-root=/srv/ftp

dhcp-boot=tag:basis,netbox-os-basis.img
dhcp-boot=tag:coronet,netbox-os-coronet.img

dhcp-range=tag:eth1,192.168.2.100,192.168.2.199,1h
dhcp-range=tag:usb0,192.168.2.100,192.168.2.199,1h
dhcp-range=tag:usb1,192.168.2.100,192.168.2.199,1h

except-interface=lo
except-interface=eth0
```

> **NOTE:** You _need_ to change the interface name in `dhcp-range`
> to match _your interface(s)_. These interfaces also need to have
> an IP address in the same subnet as used above, or change the
> DHCP range to match your interface(s).

The [dnsmasq(8)][] manual page is the authoritative documentation, so
check that for the syntax of each command.  This blog post will only
skim through the most important settings.

  - `enable-tftp` and `tftp-root` enables the built-in TFTP server,
    this is where dnsmasq expects the bootfiles to reside
  - `dhcp-host` matches the DHCP ClientID, sent by our bootloader,
    [BareBox][].  This sets the `tag`, which can be used to associate
    DHCP options to matching clients
  - `dhcp-boot` enables both DHCP option 66 (boot server) and option 67
    (bootfile) and is set to one of the two files based on what host is
    matched in the previous step
  - `dhcp-range` is the range of IP addresses to hand out to clients as
    their DHCP lease, based on the interface where the DHCP Discover is
    received on.  In our example, only three interfaces are allowed to
	serve DHCP on: `eth1`, `usb0`, and `usb1`
  - `except-interface` is probably the *most important* setting here,
    it ensures we do not accidentally start service DHCP leases on our
	home or office network ...

Enable the configuration by restarting the daemon:

    sudo systemctl restart dnsmasq.service


## Configure conserver

Modify the file `/etc/conserver/conserver.cf` to look something like this: 

```
config * {
}

access * {
        trusted 127.0.0.1;
#       trusted wkz-wmo.local;
#       trusted 192.168.0.0/16;
#       allowed 192.168.0.0/16;
}

default full {
        rw *;
}

# The '&' in logfile name is substituted with the console name.
default * {
        logfile /var/log/conserver/&.log;
        timestamp "";
        include full;
        options reinitoncc;
}

default ser {
        type device;
        device /dev/X;
        devicesubst X=cs;
        baud 115200;
        parity none;
        options !ixon,!ixoff;
        master localhost;
}

console ttyS0   { include ser; }
console ttyUSB0 { include ser; }
console ttyUSB1 { include ser; }
console ttyUSB2 { include ser; }
console ttyUSB3 { include ser; }
console ttyUSB4 { include ser; }
console ttyUSB5 { include ser; }
console ttyUSB6 { include ser; }
console ttyACM0 { include ser; }
console ttyACM1 { include ser; }
console ttyACM2 { include ser; }
console ttyACM3 { include ser; }
```

Enable the configuration by restarting the daemon:

    sudo systemctl restart conserver-server.service


## Running

With an embedded device connected to your PC, here `/dev/ttyUSB0` (see
`sudo dmesg` to find the TTY name of the device you plugged in):

    console ttyUSB0

> Use `^ec?` to get help, i.e. hold down `Ctrl` and press `e` `c` `?`

Westermo devices can load their firmware over the network if you stop
the bootloader early with `^C`, i.e. hold down `Ctrl` and press `c`
until you get to the boot menu, where you select *Shell*.  Here you
can use all [BareBox][] commands, or call our factory default scripts
to start network boot:

    boot net

This tells the bootloader to first send a DHCP Discover request to our
DHCP server (dnsmasq), which then hands out a DHCP Lease with options
66 and 67 detailing where to fetch the firmware.  The bootloader then
proceeds to connect to dnsmasq again, this time using TFTP, to fetch
the file.  When the file has been retrieved, its checksum is validated
before it is mounted by BareBox and the kernel extracted from `/boot`
inside the firmware image.  Then we're off!

[BareBox]: https://barebox.org/
[dnsmasq(8)]: (https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
