---
layout: post
title:  "FortiClient SSL VPN Silent Install with Group Policy"
description: "I've never been able to find a way to silently install the Fortinet SSLVPN client with Group Policy or otherwise. Today I had a bit of a break through."
date:   2015-10-28 21:00
tags: scripts networking windows
redirect_from:
  - /2015/10/28/forticlient-silent-install.html
---

I'm a big fan of Fortinet products; we've got a Fortigate firewall at work and it has always been completely reliable and easy (for a firewall) to configure. So when I had to implement a VPN for a handful of remote workers, I initially tried to use L2TP-IPSec which is supported by the Fortigate, but certain UK ISPs block or otherwise mess with IPSec traffic so I had to find an alternative. That alternative ended up being their proprietary SSL VPN.

The client is very simple, it's been completely reliable and the setup was extremely easy. However as more and more people have been using it, the fact I didn't have a way to silently roll it out has become a bit of a pain. I mostly use either use [Chocolatey](https://chocolatey.org) with its [Puppet Module](https://github.com/chocolatey/puppet-chocolatey) or Group Policy to push out software to Windows machines, but I couldn't find a (recent) MSI installer or a way to silently install with the EXE installer anywhere online or via their support team.

Today I had a bit of a break through. I discovered that the EXE installer creates an MSI during the installation process (although it doesn't show up if you try to extract the EXE with 7-zip or similar) which I can now deploy with Group Policy.

Heres how:

1. Download the latest installer package from [Fortinet's support portal](https://support.fortinet.com). Navigate to Download > Firmware Images. Then select Fortigate as the product and click Download. At the time of writing the latest installer can be found in /FortiGate/v5.00/5.2/5.2.4/VPN/SSLVPNTools/sslvpnclient64pkg_4.4.2317.tar.gz
2. Open up the archive with something like [7-zip](http://www.7-zip.org/) and extract SslvpnClient.exe
3. Run SslvpnClient.exe but don't click on anything in the installer
4. Navigate to C:\Users\username\AppData\Local\Temp and you'll find there is an SslvpnClient.msi that you can copy somewhere safe to deploy as normal with Group Policy.

I clearly should have read the messages that the installer spits out. I would have found this out much sooner! However I'm surprised this isn't documented anywhere online and their support team aren't aware of it.

As a little bonus, I found [this post](https://forum.fortinet.com/tm.aspx?m=96610) on the Fortinet forums. If you push out these Registry settings to HKEY_CURRENT_USER with the User Configuration > Preferences > Windows Settings > Registry part of Group Policy you can pre-configure the client and save your users some typing (and yourself some support queries).

This is the registry file that I applied to my laptop and then imported into a GPO using the Registry Wizard (easier then doing it all by hand!):

    Windows Registry Editor Version 5.00
    
    [HKEY_CURRENT_USER\SOFTWARE\Fortinet]
    @=""
    
    [HKEY_CURRENT_USER\SOFTWARE\Fortinet\SslvpnClient]
    @=""
    "KeepConnectionAlive"="1"
    "Installed"=dword:00000001
    "ConnectionName"="CompanyVPN"
    "ServerAddress"=""
    "ServerPort"=""
    
    [HKEY_CURRENT_USER\SOFTWARE\Fortinet\SslvpnClient\Tunnels]
    @=""
    
    [HKEY_CURRENT_USER\SOFTWARE\Fortinet\SslvpnClient\Tunnels\CompanyVPN]
    @=""
    "ServerCert"="1"
    "Server"="vpn.company.com:443"
    "Description"="My Company's VPN Setting"
