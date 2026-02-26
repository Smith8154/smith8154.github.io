---
title: Installing Tailscale on an Orbic RC400L
description: How to set up Rayhunter with a Telnyx SIM to receive NTFY notifications.
author: Smith8154
date: 2026-02-25 23:00:00
categories: [Personal]
tags: [Rayhunter, Cellular, Gadgets]
pin: false
math: false
mermaid: true
---

## Warnings and advisories

Just about everything I am going to show you here has the potential to brick your device, so proceed at your own risk. Make a backup of all values, and document your steps so you have something to refer back to if something goes wrong. Also note that the Orbic device itself has many known security vulnerabilities that can be exploited by a malicious carrier or anyone with access to carrier infrastructure. Connecting the device to your Tailnet has the possibility to expose the rest of your Tailnet to additional risk if a carrier were to exploit your device.

> PROCEED AT YOUR OWN RISK
{: .prompt-danger }

## Overview

In my [**previous blog post**](https://blog.wsmith.io/posts/setting-up-rayhunter/), I shared some of my thoughts and findings when setting up Rayhunter. The one thing I wanted to do in the future was get NTFY notifications working by using a Telnyx IoT SIM for super cheap cellular connectivity. If you are willing to expose your NTFY server to the public internet, this isn't the guide for you. If you want to run Tailscale on the Orbic to connect back to your selfhosted NTFY server, you have come to the right place!

In this post, I will show you how to activate a Telnyx SIM, install Tailscale on the Orbic, and install NTFY on TrueNAS. Note that I am only interested in getting Tailscale working on the device itself. I am not trying to get the clients that connect to the Orbic to also route through Tailscale. If they do, great (but probably slow), if not, sorry.

> You will need to make sure you flashed your device using `orbic-usb` as this requires a root shell. This also assumes that you have already installed ADB tools on your device.
{: .prompt-info }

## Activating the Telnyx SIM

Activating the SIM from Telnyx was a much bigger challenge than I was expecting. To the point where I thought I was going to have to give up on this dream altogether. But at the 11th hour, a beautiful person over on Github published their findings on how to [**swap the carrier**](https://gist.github.com/trevormjaymes/45ab8e596bf3a1ba858ef92e6b27909b#alternative-goahead-stock-web-api) on the Orbic. An absolutely huge thank you to Trevor for this find. This absolutely would not be possible without his amazing guide. I **highly** recommend that you go and read his full write up on this to understand more of what is going on. I'm not even going to pretend like I understand what is going on here. In short, the options in the Orbic webUI for manually setting the APN are just there for looks. They don't actually change the APN that the device is trying to connect to, so it will be stuck connecting to Verizon, no matter what SIM you insert.

> Make sure you take a backup of all of the values from this section. If you do not, there may be no way to restore these settings. Trevor is currently working on a utility to back up and restore the full firmware on the device in case you brick it. I will link to that once it is posted.
{: .prompt-danger }

> Make sure your SIM has been activated in the Telnyx portal prior to attempting this.
{: .prompt-info }

Go ahead and launch your root shell, as you will need it for all of these commands.
```shell
adb shell

/bin/rootshell
```

First, make a backup of the QCMAP file on the device.
```shell
cp /usrdata/data/qcmap/mobileap_cfg.xml /usrdata/data/qcmap/mobileap_cfg.xml.bak
```

This command will list the current PDP contexts. Save the output of this command somewhere, as you will need it to restore them if you ever wish to do so.
```shell
(cat /dev/smd7 > /tmp/at_out &); sleep 0.3; printf 'AT+CGDCONT?\r' > /dev/smd7; sleep 2; kill %1 2>/dev/null; cat /tmp/at_out
```

Next, we are going to get the current PDP contexts.
```shell
(cat /dev/smd7 > /tmp/at_out &); sleep 0.3; printf 'AT+CGDCONT?\r' > /dev/smd7; sleep 2; kill %1 2>/dev/null; cat /tmp/at_out
```

In my case, none of the contexts were what I was looking for (`data00.telnyx`), so I removed every one of them in this list. Your list should look something like this (I took this example from Trevor because I forgot to make a copy of mine):
```shell
+CGDCONT: 1,"IPV4V6","fast.t-mobile.com","0.0.0.0",0,0,0,0
+CGDCONT: 2,"IPV6","VZWAPP","0.0.0.0",0,0,0,0
+CGDCONT: 3,"IPV4V6","vzwinternet","0.0.0.0",0,0,0,0
+CGDCONT: 4,"IPV4V6","VZWAPP","0.0.0.0",0,0,0,0
+CGDCONT: 6,"IPV4V6","VZWEMERGENCY","0.0.0.0",0,0,0,0

OK
```

The number right after `+CGDCONT:` is what we are looking for next. We will remove all of these entries. Run these commands one at a time. Running too many may crash the device.
```shell
# Change out AT+CGDCONT=2 for the connections you are wanting to remove. Based on the above example, since none of them have the Telnyx connection, we would run this 5 times and change out the 2 below for 1, 2, 3, 4, and 6.
(cat /dev/smd7 > /tmp/at_out &); sleep 0.3; printf 'AT+CGDCONT=2\r' > /dev/smd7; sleep 2; kill %1 2>/dev/null; cat /tmp/at_out
```

Now, we are going to add the new PDP connection for Telnyx.
```shell
(cat /dev/smd7 > /tmp/at_out &); sleep 0.3; printf 'AT+CGDCONT=1,"IPV4V6","data00.telnyx"\r' > /dev/smd7; sleep 2; kill %1 2>/dev/null; cat /tmp/at_out
```

Next, we need to update the QCMAP and add the APN here as well.
```shell
sed -i 's|<APN>[^<]*</APN>|<APN>data00.telnyx</APN>|' /usrdata/data/qcmap/mobileap_cfg.xml
```

That should be it! Reboot your device a few times, and you should get an active data connection.

## Installing Tailscale

The first thing you will need to do is head over to [**Tailscale**](https://pkgs.tailscale.com/stable/#static) and download the latest arm binaries. After you have the tar ball downloaded, extract it. Space is very limited on the onboard flash, so it is best to extract this before copying the files over.

Before we copy the files over, we need to create a folder on the device for Tailscale's files and the socket file.
```shell
adb shell '/bin/rootshell -c "mkdir /data/tailscale && mkdir /var/run/tailscale"'
```

Now, copy the files over into the newly created folder.
```shell
adb push /path/to/extracted/files /data/tailscale
```

Next, we will set permissions on the newly copied files and create another folder for Tailscale's state data. Oddly enough, the Orbic user (UID 2000) can't write files into directories owned by its own UID, the folder has to be owned by `root`.
```shell
adb shell '/bin/rootshell -c "chown 2000:2000 /data/tailscale/* && mkdir /data/tailscale/state"'
```

Now we need to create an init script that will automatically start Tailscale when the device boots.
```shell
# Start the shell session
adb shell

# Launch the root shell
/bin/rootshell

# Create the file
vi /etc/init.d/tailscale
```

Paste the below contents into the file and save it.
```shell
#!/bin/sh

case "$1" in
  start)
    echo "Starting Tailscale..."

    # Start daemon in background
    /data/tailscale/tailscaled \
      -no-logs-no-support \
      -statedir=/data/tailscale/state &

    sleep 3

    # Bring interface up
    /data/tailscale/tailscale up --accept-routes

    ;;

  stop)
    echo "Stopping Tailscale..."
    killall tailscaled 2>/dev/null
    ;;

  restart)
    $0 stop
    sleep 2
    $0 start
    ;;
esac

exit 0
```

```shell
# While still in the root shell, change permissions on the init script
chmod 755 /etc/init.d/tailscale
```

To make it start at boot, we need to add a symlink to our init script into `rc.d`. In this case, I am going to put this in `rs5.d` so it starts later in the boot order since we want to make sure that the network is up before we attempt to start Tailscale.

```shell
ln -s /etc/init.d/tailscale /etc/rc5.d/S60tailscale
```

We need to manually authenticate to Tailscale once so it saves the authentication on the device. You will need an auth key from the portal for this.

First, we need to start `tailscaled`.
```shell
/data/tailscale/tailscaled -no-logs-no-support -tun=userspace-networking -statedir=/data/tailscale/state
```

Open a new terminal window and re-connect to the device from ADB.
```shell
adb shell

/data/tailscale/tailscale up --auth-key=<your-auth-key-here>

# Wait until your client authenticates, then disconnect
/data/tailscale/tailscale down
```

You can now close this second window, and you can quit `tailscaled` from the other ADB window. Reboot your device, and Tailscale should automatically launch and connect your device! Make sure to disable key expiry from the Tailscale portal as well so you don't have to re-authenticate in the future.

## Setting up ntfy

I initially had a bit of trouble getting push notifications to work for ntfy on iOS due to the way iOS notifications work. As it turns out, I was using a wrong environment variable. After reading the docs, and correcting my mistake, I now have notifications working through the ntfy iOS app.

Over in TrueNAS, head over to the apps page and search for ntfy and install it. Set your base URL accordingly, making sure to include the https. You will need to add two additional environment variables.
  - NTFY_BEHIND_PROXY : true
  - NTFY_UPSTREAM_BASE_URL : https://ntfy.sh

From here, configure it how you would any other Docker application on TrueNAS. I also run Nginx Proxy Manager, so I went ahead and set up the domain there for HTTPS, and added a DNS record in PiHole for my chosen domain.

In ntfy, create a topic for your Rayhinter.

## Set up ntfy in Rayhunter

You should now be able to access the Rayhunter UI via Tailscale by going to the IP of the device in your Tailnet and appending the port, for example `100.10.10.5:8080` (I don't recommend doing this often since you are paying for data per-MB). From here, put in the URL of your ntfy server and the topic, like this: `https://ntfy.yourdomain.com/rayhunter`.

## Lessons learned and future stuff

I thought this was going to be a very quick project, but this turned out to be a lot more complicated than I ever thought it would be. I had so much fun working on this, and I have learned a TON about Linux and embedded devices.

I plan to tinker with this device a bit more. My Orbic is currently running with between 2-4mb of free RAM, so I would like to see if I could get some RAM back, but we'll see. I'm sure messing with system stuff will be a good way to brick my device. I'm also going to keep an eye on the amount of data that the Orbic is using since I'm paying per-MB. I've been averaging about 20mb of data per day, which isn't too bad, but that's about $1.50 a day, which is about $1.49 more than I am wanting to pay.

If you see anything here that can be improved, please let me know by leaving a comment! If you have any questions, also leave a comment and I would be happy to help!