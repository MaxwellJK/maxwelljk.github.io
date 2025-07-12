---
layout: post
title: How to reboot a Linksys WHW01 v1 running OpenWRT
description: >
  Failsafe mode
sitemap: false
hide_last_modified: true
---

That's an easy one!

Unplug the router and leave it disconnected for a few seconds. Disconnect also the WAN cable, to avoid issues later on.

When the LED starts blinking, it is time to press the reset button at the bottom for a few seconds.
After a few seconds, the LED will stop blinking and will be completely off: it is now possible to connect the laptop to the router using a LAN cable and connect to the router by running:

~~~bash
ssh 192.168.1.1
~~~

If your laptop is not acquiring an IP address, you will have to force it by manually setting `192.168.1.2` as IP, `255.255.255.0` as subnet and `192.168.1.1` as gateway.
{:.note}

If your IP is not `192.168.1.x`, adjust all the commands accordingly.
{:.note.smaller}

SSHing into the router is what worked for me. Somewhere else i've read TELNETing works(e.g. `telnet 192.168.1.1`)

If the connection is esablished correctly, we are in failsafe mode.

Running the below, will allow to remount the root partition from read-only to read/write mode
~~~bash
mount_root 
~~~

Once the root is mounted, we can operate `opkg` or any other command we need to.
This mode is particularly useful when fixing issues caused while playing with Tailscale (coff coff!!)

In the worst case scenarion running the below will reset the router to the original state (and avoid reflashing).

~~~bash
firstboot 
~~~

Once done, a nice reboot will restart the router
~~~bash
reboot -f ## -f forces the reboot 
~~~

Happy fixing!