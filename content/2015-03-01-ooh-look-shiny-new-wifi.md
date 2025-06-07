---
layout: post
title:  "Ooh look! Shiny new Wi-Fi!"
description: "The access points powering the guest and internal Wi-Fi networks at work were getting a bit old and senile, so around 6 months ago we decided to get some money to upgrade. At the time we only thought only one needed replacing, so we went out and bought a Cisco WAP321 to replace it; having prior knowledge of their reputation and not wanting to be burdened..."
date:   2015-03-01 18:35
tags: hardware networking
redirect_from:
  - /2015/03/01/ooh-look-shiny-new-wifi.html
---
The access points powering the guest and internal Wi-Fi networks at work were getting a bit old and senile, so around 6 months ago we decided to get some money to upgrade. At the time we only thought only one needed replacing, so we went out and bought a [Cisco WAP321][wap321] to replace it; having prior knowledge of their reputation and not wanting to be burdened with another unreliable access point. However we've since realised that we needed a second access point, so this past week we bought a [Ubiquiti Unifi AP-AC][ap-ac] which I'll compare to the Cisco.

**The Cisco**

Overall the setup was painless and the web interface works in any modern browser, unlike the Cisco SGE2010 switches we have, which require Internet Explorer in Compatibility Mode. The only problem I had was unfortunately quite a major one; by default the access point has "Spanning Tree Mode" enabled. Logically you would think that as we use use STP on our 3 SGE2010 switches, this would be just fine. Sadly it put the switches into an STP loop and more or less brought down the entire LAN. Not very helpful at all... This problem persisted regardless of changes to switch port configuration until I found and turned off the option on the access point. It's under Wireless > WDS Bridge on the web interface if you're having the same problem.

![Cisco Hardware](/media/wifi2.jpg)

The web interface, despite being very plain and Cisco-ish provides lots of useful functionality. Features that jumped out at me include packet capture, captive portal, remote syslog, SNMP and rogue AP detection. This is in no way a complete list and I can't vouch for half of them as I just set it up for our two networks; our internal network using WPA2 Enterprise talking back to a Microsoft Network Policy Server via RADIUS on one VLAN and our guest network using WPA2 Personal on another VLAN. All the heavy lifting to lock down the guest network is done on our Fortigate firewall, although I could offload a lot of it to this access point if I wanted to.

![Cisco Interface](/media/wifi1.png)

Since I configured it, the WAP321 has worked faultlessly and provided great performance. Although I wish it could broadcast on both 2.4GHz and 5GHz bands simultaneously like its bigger brother the WAP371; a model which is perhaps more comparable to the Unifi access point I'll move onto shortly.

**The Unifi**

So, a few months later we've got problems with the Wi-Fi at the other end of the office and since buying the Cisco I've taken to reading [/r/sysadmin] on my OnePlus One before I drag myself out of bed in the morning. On this most excellent community I've seen numerous mentions of Ubiquiti's products and I don't think any of them have been negative; so after some further research I decided to recommend we got a [UniFi AP-AC][ap-ac] instead of another Cisco. 

Although the good reviews helped, the decision was mostly based on 2 things; firstly that £200 would get us another WAP321 and PoE injector from Cisco which is only single band (at a time) and 802.11n, whereas the AP-AC is dual band and supports the new 802.11ac standard. The second major factor was that the WAP321 will only support 32 clients at a time[^ciscodatasheet] (although I'm not sure what exactly happens when you hit that) which doesn't leave us much room for expansion, whereas the Unifi supports 200+[^unifidatasheet]. That's a lot of extra bang for our precious IT budget buck!

![Unifi Hardware](/media/wifi3.jpg)

The Ubiquiti UniFi AP-AC impressed straight out of the box; unlike the Cisco it ships with a Gigabit PoE injector (instead of being £50 extra), it also comes with some very sturdy looking wall mounting brackets and all the screws you need. I even had comments when I was configuring it at my desk about just how smart it looked, with the subtle blue light that illuminates when it is up and running. 

Unlike the Cisco it doesn't have a built-in web interface (although it does have an SSH interface); instead you need to install their very smart UniFi Controller application on a server. The download page only lists Mac and Windows versions so I put it on a Windows Server 2012 R2 VM I had, but it turns out there is a Linux version too which you can download from the [UniFi Updates Blog][unifi-blog]; just make sure you've got a Java JRE installed and ports 8080 and 8443 aren't used.

The first machine I installed the Windows software on had something else listening on port 8080, but instead of bringing up an error, the software just listened on localhost and of course the access point couldn't connect to it. So after working this out, I put the software on a different machine and waited patiently for a few minutes, but the access point still didn't pop up in the web interface. After a bit of Googling I found these instructions [on the Spiceworks Community][spiceworks] which solved the problem and got me to the point where I could start playing with the many configuration options. Yay!

Like the hardware, the (flash-based) web interface gives a great first impression; after logging in you are presented with an example office floorplan, invited to upload your own and then drag your access points to their location. Unfortunately, despite desperately wanting more access points, so I can play with all the functionality properly, I've only got the one; so I completely skipped this tab (for now at least) and headed straight for the Settings menu to set up and set it up to mirror the Cisco with our two networks.

There is lots of functionality which mirrors the Cisco; it also has captive portal, remote syslog, SNMP and rogue AP detection as well as numerous settings for things like locking down guest access or bandwidth limiting a group of users. It is missing packet capture, although tcpdump comes pre-installed and is accessible via SSH.

For me, other than the extra performance and capacity you get for your money, the very flashy charts and tables it gives you are the biggest selling point. You can see lots of data about each user that is currently connected to your access point(s) and also lots of metrics about usage over time.

![Useful data](/media/wifi4.png)

![Shiny charts](/media/wifi5.png)

On the face of it, this might just look like a lot of pointless graphs that you can print out and bore management with. However the way they've implemented this makes it very easy to take action based on the data you're given, such as blocking or rate limiting a user who is hogging your bandwidth. This reduces troubleshooting time and any SysAdmin will tell you that this is worth its weight in gold!

**Conclusion**

The Cisco WAP321 is a very reliable access point, with a lot of functionality. If I was told I could only use Cisco access points for the next 10 years, I wouldn't be particularly upset because I'm sure they would give me very few problems. However the Unifi AP-AC does everything the Cisco does, with much better performance for the same amount of money. It also gives you a very powerful centralised management interface which allows you to easily scale up to much bigger numbers of access points and clients. Unless something changes drastically over the next few months I will definitely be recommending these to anyone, regardless of the deployment size.




[wap321]: http://www.cisco.com/c/en/us/products/wireless/wap321-wireless-n-selectable-band-access-point-single-point-setup/index.html
[ap-ac]: https://www.ubnt.com/unifi/unifi-ap-ac/
[unifi-blog]: http://community.ubnt.com/t5/UniFi-Updates-Blog/bg-p/Blog_UniFi
[/r/sysadmin]: http://www.reddit.com/r/sysadmin
[^ciscodatasheet]: http://www.cisco.com/c/dam/en/us/products/collateral/wireless/wap321-wireless-n-selectable-band-access-point-single-point-setup/aag_c45-717569.pdf
[^unifidatasheet]: http://dl.ubnt.com/datasheets/unifi/UniFi_AP_DS.pdf
[spiceworks]: http://community.spiceworks.com/how_to/9692-manually-setting-the-controller-address-for-a-unifi-ap
