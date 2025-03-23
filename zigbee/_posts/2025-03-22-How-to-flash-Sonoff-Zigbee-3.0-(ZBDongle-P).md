---
layout: post
title: How to flash Sonoff Zigbee 3.0 (ZBDongle-P) USB firmware using a Mac
description: >
  How to flash a zigbee dongle in 10 mins (or less)
sitemap: false
hide_last_modified: true
---

During one of my "it would be cool to have that in my home automation setup" moment, I started pushing my limits and exploring uncharted territory.
This is when the unthinkable happened: my zigbee dongle was unresponsive.

Not only it was unresponsive, I managed to make it unusable!

![Screenshot](/assets/img/blog/zigbee/nvram_corrupted.png){:.lead loading="lazy"}

A quick research showed it is a common issue and easily fixable: reflashing the firmware, as the error message already suggested.
Not a problem I think, I have flashed other devices in the past and I just needed to find the right tools and instructions. Unfortunately all the websites I found were suggesting to use a tool for Windows while i only owwed a mac.

Took me a while but then I found a reddit post pointing to a website mentioning [another website](https://vandalen.dev/post/how-to-update-sonoff-zigbee-3-0-zbdongle-p-usb-firmware-using-a-mac/), which honestly saved me.

I am not here to take any credit for this. I just found it but i will also forget what the solution is so I am saving myself the hassle of looking for this wesite again.
{:.note}

###### Python requirements
On Macs python is already installed, so I am skipping this step.

~~~bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install pyserial intelhex
~~~

###### cc2538-bsl
cc2538-bsl is a python script that communicates with the boot loader of the Texas Instruments CC2538, CC26xx and CC13xx SoCs (System on Chips). It can be used to erase, program, verify and read the flash of those SoCs with a simple USB to serial converter.

~~~bash
mkdir cc2538-bsl
cd cc2538-bsl
curl -sSL https://github.com/JelmerT/cc2538-bsl/archive/refs/heads/master.tar.gz | tar xz --strip 1
~~~

###### Download and flash the firmware
For the latest version, you may want to check [this repository](https://github.com/Koenkk/Z-Stack-firmware/tree/master/coordinator/Z-Stack_3.x.0/bin).

Find the usb device by checking the connected usb tty devices like so:

~~~bash
ls /dev/tty* | grep usb
~~~

Download and extract the firmware into the current directory

~~~bash
wget https://github.com/Koenkk/Z-Stack-firmware/raw/master/coordinator/Z-Stack_3.x.0/bin/CC1352P2_CC2652P_launchpad_coordinator_YYYYMMDD.zip
unzip CC1352P2_CC2652P_launchpad_coordinator_YYYYMMDD.zip

python3 cc2538-bsl.py -ewv -p /dev/tty.usbserial-0001 --bootloader-sonoff-usb ./CC1352P2_CC2652P_launchpad_coordinator_YYYYMMDD.hex
~~~

Replace the value of `YYYYMMDD` with the version of firmeware you want to flash. At the time of writing, it is `20240710`.
{:.note.smaller}

Replace the `/dev/tty.usbserial-0001` with the one identified above.
{:.note.smaller}

`-ewv` means Mass erase, write, verify
`-p` is the port on which your device is running, in this case /dev/tty.usbserial-0001
`--bootloader-sonoff-usb` means that the bootloader is activated by the script, by toggeling RTS and DTR in the correct pattern for Sonoff USB dongle (remove this if your device is not a Sonoff dongle).

The output should look like

~~~bash
Opening port /dev/ttyUSB0, baud 500000
Reading data from ../CC1352P2_CC2652P_launchpad_coordinator_20240710.hex
Your firmware looks like an Intel Hex file
Connecting to target...
CC1350 PG2.0 (7x7mm): 352KB Flash, 20KB SRAM, CCFG.BL_CONFIG at 0x00057FD8
Primary IEEE Address: 00:00:00:00:00:00:00:00
    Performing mass erase
Erasing all main bank flash sectors
    Erase done
Writing 360448 bytes starting at address 0x00000000
Write 104 bytes at 0x00057F988
    Write done
Verifying by comparing CRC32 calculations.
    Verified (match: 0xe0c256fd)
~~~

Sonoff dongle succesfully flashed!