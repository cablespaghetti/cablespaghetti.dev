---
layout: post
title:  "Hosting a Fediverse instance on an original Raspberry Pi"
description: "Hosting a Fediverse instance site on an original 256MB Raspberry Pi using Alpine Linux diskless mode."
date: 2025-06-19 22:00
tags: hardware, raspberrypi, alpinelinux, linux
banner_image: media/pi-with-hdd.jpg
---

In my previous post, I moved my blog to the original 256MB RAM Raspberry Pi that I purchased for the princely sum of £25 in May 2012. When I'd finished that little project I couldn't help but notice that due to not being a particularly successful blogger, I still had a lot of resources to spare.

I had heard of [snac](https://codeberg.org/grunfink/snac2) from following [Justine Smithies](https://snac.smithies.me.uk/justine) who hosts her Fediverse presence on it. I knew it was stupidly lightweight, even more so than [GoToSocial](https://gotosocial.org/) and quickly discovered that some helpful individual has even [packaged it for Alpine Linux](https://pkgs.alpinelinux.org/package/edge/community/armhf/snac) with armhf/armv6 builds available!

Whilst I do like snac and am currently using it as primary fedi instance in place of Mastodon, I will preface this with the warning that it is *very* minimal, and definitely aimed at UNIXy sysadmin types. Do not expect the full feature set that you get with Mastodon, and compatibility with Mastodon apps is patchy at best in my experience (although I have got [Mona](https://apps.apple.com/us/app/mona-for-mastodon/id1659154653) for iOS to work quite well).

## Installation

As snac is still under heavy development, I wanted the newest possible version, so whilst I am running the latest stable Alpine Linux, I wanted the version of snac from the "edge" repository. To achieve this I just pointed `/etc/apk/repositories` at the edge version of the community repository:

```
/media/mmcblk0p1/apks
http://dl-cdn.alpinelinux.org/alpine/v3.22/main
http://dl-cdn.alpinelinux.org/alpine/edge/community
```

Then it's as simple as `apk add snac`. I also ran `rc-status add snac default` to make sure it starts up on boot.

## Storage

"But Sam!" I hear you say, "The photo at the top of the post shows a 2.5" hard disk!"

Yes, well spotted. Lightweight or not, I don't want to store all my snac data on an SD Card. So consistent with the theme of this series, I found the oldest and slowest 2.5" hard disk (an IDE 40GB Fujitsu from 2004), random USB adapter and a powered USB 2.0 hub to connect up to my terrible server.

![Raspberry Pi with a hard disk connected via USB and a powered USB hub](/media/pi-with-hdd.jpg)

When connected it showed up in `dmesg` as `/dev/sda`, so I ran `apk add xfsprogs` to get the XFS utilities, used `fdisk /dev/sda` to make a new MBR partition table and single "Linux" (the default) type partition on it and `mkfs.xfs /dev/sda1` to format the partition.

Amazingly considering the age of the drive and the amount of time it has been sat in my pile of mostly dead hard drives, this all worked properly and I could `mount -t xfs /dev/sda1 /var/lib/snac`. I then added this line to `/etc/fstab` so it would mount on boot:
```
/dev/sda1       /var/lib/snac   xfs     defaults 0 2
```

We'll get to backups later; because you are definitely going to want backups with a setup like this...

## Snac configuration

It was at this point at the snac [System Manager's Manual](https://comam.es/snac-doc/snac.8.html#Data_storage_Initialization) became very useful. Usually you would have the man pages available locally, but I don't...because lightweight...

For the Alpine package the `snac init` script mentioned in the above documentation is automatically triggered when you run `rc-service snac start`. Please read the docs linked above, but my answers looked like this:

```
Network address or full path to unix socket [127.0.0.1]: 
Network port [8001]: 
Host name: cablespaghetti.dev
URL prefix: /fedi
Admin email address (optional): myemail@example.com
```

As you can see I used the defaults for the networking stuff, gave it my domain name and told it that snac is going to live at `/fedi` because this blog lives at the root. It then started up and everything was rosy. You can edit the other settings such as the server description in `/var/lib/snac/data/server.json` and then run `rc-service snac restart`.

You'll want to add your user at this point, which I did by running `su -s /bin/sh - snac` to get a shell as the `snac` user, then `snac adduser /var/lib/snac/data`.