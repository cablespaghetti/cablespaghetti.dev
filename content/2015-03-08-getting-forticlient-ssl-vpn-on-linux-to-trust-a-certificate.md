---
layout: post
title:  "Getting FortiClient SSL VPN on Linux to trust a certificate"
description: "I'm a Linux guy; I find it to be the most intuitive operating system for most tasks, even on a Laptop. At home I use Arch Linux and at work I've recently moved from the Windows 7 workstation I've been using for the last 10 months to Ubuntu GNOME on an old laptop that was feeling unwanted."
date:   2015-03-08 16:34
tags: linux networking
redirect_from:
  - /2015/03/08/getting-forticlient-ssl-vpn-on-linux-to-trust-a-certificate.html
---
I'm a Linux guy; I find it to be the most intuitive operating system for most tasks, even on a Laptop. At home I use [Arch Linux](https://www.archlinux.org) and at work I've recently moved from the Windows 7 workstation I've been using for the last 10 months to [Ubuntu GNOME](http://ubuntugnome.org) on an old laptop that was feeling unwanted. A couple of the developers have also made the move, although they're currently using the default Unity desktop which I can't stand.

Anyway, as usual I digress. We use [StartSSL](https://www.startssl.com/) certificates for our external services, because why pay for SSL certs when you can have them for free? These certs are trusted by pretty much everything, but for some reason the Linux version of Fortinet's SSL VPN client doesn't. As there is no way to turn off certificate trust checking (which is a bad idea anyway) I couldn't connect to the VPN from either my work Ubuntu laptop or home Arch Linux laptop. It took me a while to diagnose as the software just hangs on 'Connecting...', but the forticlientsslvpn.log file in the helper directory within the FortiClient install directory helped.

The only documentation I could find on this problem was on the website of the [University of Bamberg](http://www.uni-bamberg.de/fileadmin/rz/vpn/Linux/VPN_Linux_FortiClient.pdf) in Germany. As this is in German it is pretty hard to find (although straightforward to follow).

Here's what you need to do after you have installed the VPN client (this is well documented elsewhere so I'll leave it out):

1. Make a .fctsslvpn_trustca directory in your home directory.
2. Save your CA's root certificate in this folder in Base-64 Encoded PEM format ([StartCom's is available here](http://www.startssl.com/certs/ca.pem)).
3. Start (or restart) FortiClient and your VPN should now connect.
