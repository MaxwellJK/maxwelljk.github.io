---
layout: post
title: Running ZHA and Z2M at the same time with one single USB dongle (Sonoff Zigbee 3.0 P)
description: >
  a.k.a. how to make my life miserable
sitemap: false
hide_last_modified: true
---

Internet is always full of interesting articles that can inspire others. 
Such inspirations can come completely unexpected - but each situation is different so it's not always possible to replicate step by step what was presented to us...and sometimes a certain amount of creativity is needed.

This time what inspired me was an article which mentioned how, with a specific Zigbee dongle, the author was able to connect both ZHA and Z2M to their dongle and make use of both interfaces for test purposes.
I fancied that, but it wasn't possible. Or was it?

## Stating the problems

To start with, my device was different - theirs was able to connect to the network via ethernet port - mine is connected to USB.
Second, I didn't need it for test purposes, mine was an actual need - a [zigbee switch](https://moeshouse.com/products/zigbee-wireless-self-powered-scene-switch?srsltid=AfmBOoowN9oyVflEjpEAbNJ4ATrIt3V-L45Hy6gR23COSmIwuu8UTP-o) was not compatible yet with ZHA but it was [under development](https://github.com/Koenkk/zigbee2mqtt/issues/19405) in Z2M, with promising results.

I needed Z2M without saying NO to ZHA because I like it and because all my integrations were already running fine and didn't want to pair all the devices again and change all my automations (spoiler: I had to pair all my devices again...but I managed to save the automations)

Given this problem, I started where I always start: a google search - apparently without success since it seems nobody ever attempted it with a USB device.
So I tried running Z2M while also running ZHA and I managed to cause myself quite a lot of troubles (including an [unusable dongle](https://maxwelljk.github.io/zigbee/2025-03-22-How-to-flash-Sonoff-Zigbee-3.0-(ZBDongle-P)/)).
It makes sense, USB devices are never able to serve 2 consumers at the same time. Either ZHA or Z2M, it can't serve both.

I needed an extra layer in between, something that could accept multiple requests while managing the interactions with a single USB.
Again, the answer to this problem happened to be in a different blog and its name was `Ser2Net`.

## Ser2Net

Ser2Net is a program that allows to access serial ports over a network connection, effectively acting as a bridge between serial devices and TCP/IP networks.
It was clear, this is what I needed but before I had to make sure ZHA and Z2M could handle remote dongles - it was an easy check and both could.
The solution was closer but not that close yet. I had all the pieces, just needed to put them together in a nice way.

## Working on a solution

Step zero was stopping ZHA and Z2M to make sure nothing was actually accessing and using the dongle.

Given that Ser2Net allows inbound connections via TCP/IP networks, the obvious next step was to deploy a docker container in my lab network, making sure the USB was mounted.
Ser2Net also needs a configuration file and possibly the most basic one may look like this:

~~~bash
connection: &con01                          #name of the connection
    accepter: tcp,20108                     #protocol and port
    enable: on                              #connection enabled or not
    options:                                
      kickolduser: false                    #disconnect old users
      max-connections: 2                    #number of max inbound connections
    connector: serialdev,                   #type of connector (serial device)
              /dev/ttyUSB0,                 #device path
              115200n81,                    #speed
              nobreak,                      #disables automatic clearing of the break setting,
              local                         #local connections only
~~~

Once that was stored on my server, it was time to spin up the container.

I am skipping all the network configuration in docker.
{:.note}

~~~ yml
services:
  ser2net:
    container_name: ser2net
    hostname: ser2net
    image: ghcr.io/jippi/docker-ser2net
    restart: unless-stopped
    ports:
      - 20108:20108
    volumes:
      - /volume1/docker/Ser2Net/ser2net.yaml:/etc/ser2net/ser2net.yaml
    devices:
      - /dev/ttyUSB0
~~~

No errors - great.

Following the instructions on both ZHA and Z2M websites, I tried to connect them to Ser2Net singularly and it was working pretty well but when both were up and running, ZHA and Z2M were conflicting with each other...which sounded odd to me considering the access to the dongle was via TCP/IP.

A quick look at the error logs and I started realising how each one of them was writing some configuration onto the device (namely `PAN ID`, `Extended PAN ID` and `Network Key`).
It took some trial and error to actually identified the parameters but after that the solution was once again one click away.

While ZHA doesn't allow any customisation for these 3 parameters, Z2M accepts static values in its config file. This is also documented on the [Z2M website](https://www.zigbee2mqtt.io/guide/configuration/zigbee-network.html#network-config).

At this point it was simple: ZHA had to be executed first, so that the device could be configured without issues.
![Screenshot](/assets/img/blog/zigbee/zha_network_settings.png){:.lead loading="lazy"}

The Network key is not visible in the screenshot above. Downloading the backup will help.
{:.note}

After this was done, Z2M had to be configured with the same values from ZHA.

It's been running extremely fine more than 2 months now and I can say I am expremely happy.


### Disclaimer
Although Ser2Net does the job extremely well, it _may_ not always be suitable since it is adding an extra layer and communications may be affected (extra latency).