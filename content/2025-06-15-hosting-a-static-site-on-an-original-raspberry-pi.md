---
layout: post
title:  "Hosting a static site on an original Raspberry Pi"
description: "Hosting a static site (this blog) on an original 256MB Raspberry Pi using Alpine Linux disless mode."
date: 2025-06-15 22:00
tags: hardware raspberrypi alpinelinux linux
image: assets/raspberrypiimage[asgla[gpals]]
---

It's been a while since I posted to this blog, but I'm making another attempt at getting back in the habit of writing things so here we go.

For a long time, I've had a strange obsession with making use of the worst possible hardware to do "stuff". This is despite being lucky enough to be able to afford much better and in fact having a collection of semi-decent random computers sat around doing nothing.

Thanks to my friend [Phill](https://thega.me.uk/), who does not share the same hoarding tendecies as myself, as well as the semi-decent hardware, I also have a collection of first generation Raspberry Pis, two from 2011 with 256MB RAM and a slightly later one with 512MB. Naturally I decided that the best way to resurrect this blog would be to move to from GitHub Pages to one of these boards.

I have attempting to make use of this boards in the past, but the limiting factor always ends up being storage. They are only capable of booting from an SD Card and your average Linux-based workload both runs like treacle and eventually kills this very sub-optimal storage medium.

The solution to this is [Alpine Linux "diskless" mode](https://wiki.alpinelinux.org/wiki/Diskless_Mode); this runs the whole OS, applications and any configuration you need to persist from RAM! Yes, I only have 256MB of RAM to work with but at least whatever I can fit into this space won't be bottlenecked by terrible storage performance.

# Setting up the SD Card

Alpine Linux has some [great documentation on how to install on a Raspberry Pi](https://wiki.alpinelinux.org/wiki/Raspberry_Pi), and are one of the few distros still supporting the 32bit ARMv6 processor in these old Pis. However I will document the process I followed here, because I did hit a number of issues.

The first issue, was that the Raspberry Pi Imager route creates an absolutely tiny partition which seem to use FAT16. A bug in dosfstools means I found no way to increase this to a workable size under Linux. So I followed the Manual method mentioned on the wiki page.

With the SD card still connected to my laptop I then completely ignored the note at the top of config.txt and changed the contents to the following. This is because certain settings such as `gpu_mem=16` do not work if placed in usercfg.txt and I thought I'd just make all my changes in the same place. The most important change here is reducing the amount of memory allocated to the GPU to maximise the amount of RAM available for activities; without this change I could not get the Alpine Linux installation script to complete. I have also overclocked the snot out of the CPU (your mileage may vary) which is somehow completely stable without so much as a heatsink and running at just over 50C under load.

```
# do not modify this file as it will be overwritten on upgrade.  
# create and/or modify usercfg.txt instead.  
# https://www.raspberrypi.com/documentation/computers/config_txt.html  
<br/>kernel=boot/vmlinuz-rpi  
initramfs boot/initramfs-rpi  
arm_64bit=0  
gpu_mem=16  
arm_freq=1100  
core_freq=500  
sdram_freq=500  
over_voltage=8  
include usercfg.txt
```

Before you unmount your SD Card you need to grab `fixup4cd.dat`, `fixup_cd.dat`, `start_cd.elf` and `start4cd.elf` and place them alongside the rest of the firmware files. These are the cut down firmware files which get used when you set `gpu_mem=16` which are not shipped by Alpine Linux by default. I think you might be able to go without the "4" ones as I suspect they are for the Pi 4/5 but I copied them anyway.

