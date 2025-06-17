---
layout: post
title:  "Revenge of the flaky routers"
description: "There is an DSL router at work which I'm sure is sentient. It's 100% reliable for months on end and then once in a while, always when I'm out of the office, it decides to crash and drop the Internet connection. The only solution is of course to turn it off and on again; which means visiting the office at the most inconvenient times possible."
date:   2015-02-23 19:03
tags: scripts, linux, networking
redirect_from:
  - /2015/02/23/revenge-of-the-flaky-routers.html
---
There is an DSL router at work which I'm sure is sentient. It's 100% reliable for months on end and then once in a while, always when I'm out of the office, it decides to crash and drop the Internet connection. The only solution is of course to turn it off and on again; which means visiting the office at the most inconvenient times possible.

I would spend some money from the IT budget to get it replaced but with us moving to a leased line in the near future and it only failing every few months, it's really not worth it. Being a sysadmin, the natural solution was to write a shell script.

**Disclaimer: I am not a shell script guru. I take no responsibility for any exploding routers or other hilarious events that may transpire as a result of the use of these scripts.**

The first part of the script does the actual logging in and rebooting. I must admit I had no idea how to do this part, but found the solution in this [nixcraft forum thread][nixcraft]. It uses `expect` so make sure this is installed on your system. I saved it as `reboot.sh`, but call it whatever you like, just remember to change the reference to it in the next script.
```sh
#!/usr/bin/expect
set timeout 20

spawn telnet 1.3.3.7
expect "Password:"
send "supersecurepassword\r"
expect "ras> "
send "sys reboot"
send "\r"
expect "ras> "
send "exit"
send "\r"
sleep 5
exit
```

This second part runs in an infinite loop (the good kind, honest!), checking it can reach an IP address on the Internet every 5 seconds (I recommend using your ISP's next hop router). If it can't then it retries for at least 2 minutes (usually around 3-4) and then runs the router reboot script and sleeps for 10 minutes before doing anything else.
```sh
#!/bin/bash
message="Insert super informative email message here."
 
while true; do
    count=120
    while [ $count -ne 0 ] ; do
        sleep 1                                #Wait at least 1 second between checks
        ping -W 1 -c 1 8.8.8.8 >& /dev/null    #Check connectivity
        rc=$?
        if [ $rc -ne 0 ] ; then
            let count=count-1                  #If the ping failed decrement count
        else
            count=120                          #Otherwise make sure count is 120
            sleep 5                            #Wait 5 seconds before checking again
        fi
    done
 
    echo "No ping response from Internet connection for over 2 minutes. Rebooting router and sleeping for 10 minutes." #Log message
    echo $message | mail -s "Router Reboot Alert" email@address.com #Send email
    ./reboot.sh                                #Reboot router
    sleep 600                                  #Go to sleep for 10 minutes
done
```

If you happen to be using a systemd based Linux distribution like CentOS 7, here's a bonus part for you to get the script running as a service. I saved this as `/usr/lib/systemd/system/routerreboot.service` then ran `systemctl start routerreboot.service` to start it and `systemctl enable routerreboot.service` to make sure it starts after a reboot of the server.
```
[Unit]
Description=Start router reboot script

[Service]
User=preferablynotroot
ExecStart=/script/path/script.sh
KillMode=process

[Install]
WantedBy=multi-user.target
```

[nixcraft]: http://nixcraft.com/showthread.php/15060
