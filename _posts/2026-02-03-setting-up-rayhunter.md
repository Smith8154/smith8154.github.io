---
title: Setting up Rayhunter
description: Learnings after installing Rayhunter.
author: Will Smith
date: 2026-02-03 23:00:00
categories: [Personal]
tags: [Rayhunter, Cellular]
pin: true
math: true
mermaid: true
---

## What is Rayhunter?

I won't go into too much detail here as I'm probably not the most qualified person to give a lecture on this. Simply put, Rayhunter is [**software developed by the EFF**](https://www.eff.org/deeplinks/2025/03/meet-rayhunter-new-open-source-tool-eff-detect-cellular-spying) to try and detect the use of Stingry's or other IMSI catchers. These devices are used by law enforcement to track mobile device, often without a warrant (which is a 4th amendment violation). Not much is known about how these devices truly operate, which makes them difficult to detect. If you end up setting this up yourself and your device alerts you to potential activity, make sure and [**submit it to the EFF for analysis**](https://efforg.github.io/rayhunter/faq.html#help-rayhunters-line-is-redorangeyellowdotteddashed-what-should-i-do).

Below are some excellent videos from some of my favorite creators covering the topic.
{% include embed/youtube.html id='W_F4rEaRduk' %}
{% include embed/youtube.html id='UtwQ9APzYWA' %}

## Purchasing a device

I chose to purchase the Orbic RC400L since this is the device recommended by the EFF. I purchased my device off ebay for about \$30 shipped. While you can purchase units with Rayhunter preinstalled, wheres the fun in that? I noticed that most of the units being sold with Rayhunter preinstalled were over \$50. Installing Rayhunter is dead simple, so I really don't think it's worth purchasing these devices for such a massive upcharge when installing Rayhunter only takes about 5 minutes.

Wherever you decide to purchase your device from, make sure it includes a SIM card as that is a requirement for Rayhunter to operate, even if you don't activate it. You can purchase a SIM card separately, but I think it makes sense to purchase it with the unit.

### Locked vs unlocked

It appears that some Orbic devices are being sold with a carrier lock (usually locked to Verizon). It is very difficult to tell which devices are carrier locked without trying to put a different SIM card in it to see if it works. The good news is the carrier lock doesn't matter for Rayhunter operations. It would only matter if you actually wanted to activate the SIM to use data or if your device was sold without a SIM card and you want to use a SIM card from a different carrier that you may have already laying around.

## Software versions

For installing Rayhunter, there doesn't appear to be a software version requirement as the EFF isn't exploiting a vulnerability on the device to install Rayhunter, rather it is making use of Android tools installed on the device. That said, I found [**one single article**](https://support.cs.inc/s/article/115003355127-Supported-Wi-Fi-mobile-hotspots-for-USB-tethering) that mentions that USB tethering only works on software version 1.2.1 and newer. Naturally, my device shipped with version 1.2.0 and the only way to update the firmware is to have an active cellular connection. Verizon has a [**support article**](https://www.verizon.com/support/verizon-orbic-speed-mobile-hotspot-update-instructions/) for updating the firmware using a Windows-only program, so I decided to give that a try. I booted into my Windows partition and downloaded the program (you get a cert error when visiting the download page, which is just great), but it failed to update my device. When you open the program, it specifically mentions it is for upgrading devices with firmware version 1.1.5, so I guess it really only works with version 1.1.5 and nothing else, or I am just very unlucky. I am still trying to figure out if there is a way to upgrade the firmware without a cellular connection, but so far I haven't been able to find anything.

## Installing Rayhunter

For this, I will refer you back to the [**EFF's guide**](https://efforg.github.io/rayhunter/installing-from-release.html), as the install instructions may change over time. My only recommendation would be to install Rayhunter using the `orbic-usb` method so you can have a root ADB shell on the device.

> To change between install methods (`orbic` and `orbic-usb`), you must first uninstall Rayhunter, then reinstall. If you first installed Rayhunter with `orbic` and you now want a root ADB shell and you try and run `orbic-usb`, you will get an error.
{: .prompt-info }

## Extra stuff

Congrats! You now have Rayhunter installed, and you are ready to go out and start searing for IMSI catchers! I have some more goodies below for things that I found useful after getting Rayhunter installed.

### ADB

ADB, or ***Android Debug Bridge***, is a CLI tool for communicating with Android devices. For the Orbic, it will allow us to gain shell access to the device. If you chose to install Rayhunter with `orbic-usb`, you will be able to gain root access as this install method also installs `/bin/rootshell`, which you can use to drop into a root shell on the device.

On macOS, you can install ADB via [**Homebrew.**](https://brew.sh)
```bash
brew install android-platform-tools
```

Now that you have ADB installed, it's time to connect to your device.
```bash
# Kill any existing connections
adb kill-server

# Start a shell session
adb shell
```

> Keep in mind that if you chose to flash your device using `orbic-usb`, you will need to [**re-enable USB tethering.**](https://efforg.github.io/rayhunter/faq.html#how-do-i-re-enable-usb-tethering-after-installing-rayhunter_)
{: .prompt-info }

### Disabling WiFi

In my case, I don't care to have the device broadcasting a WiFi signal 24x7. I didn't want to contribute to WiFi pollution and it's not a feature I ever plan to use since I can see the status by just looking at the screen. I also figure it will save a bit of battery as well, though I haven't tested this. If you want to be paranoid, the WiFi signal the device broadcasts could easily be used to track you as well. Since I already have a way to access the webUI over USB, I still have a way to transfer any files back and forth, make configuration changes, and update the device.

> You must have flashed your device with ADB mode to disable WiFi, as it requires root access.
{: .prompt-info }

Follow the [**EFF guide**](https://efforg.github.io/rayhunter/faq.html#how-do-i-disable-the-wifi-hotspot-on-the-orbic-rc400l) to disable WiFi.

Now that you have WiFi disabled, if you want to access the webUI, you will need to use ADB to forward the port from the Orbic to your local machine.
```bash
# Kill any existing connections
adb kill-server

# Tunnel port 8080 from the Orbic to port 8080 on your device. If port 8080 is already in use on your device, change the first port to something else
adb forward tcp:8080 tcp:8080
```

From here, the Rayhunter webUI will be accessible at `localhost:8080` on your device.

### Future stuff

Rayhunter has [**ntfy**](https://ntfy.sh) built in, but it requires an active cellular connection to use (and it [**may or may not work with iOS**](https://github.com/EFForg/rayhunter/issues/770#issuecomment-3740680849)). I am interested in paying absolute bottom-dollar to get this feature to work, so I have been looking into using an IoT SIM since I would be using essentially zero data, and most of these IoT SIM's have you pay per-MB, which could add up quickly. I have ordered a SIM from [**Telnyx**](https://telnyx.com) to give this a try, as they have some of the cheapest rates I have seen. The SIM is still in the mail, so I will either update this post or make a new post when I get the SIM in and am able to test it. At the very least, it will allow me to update the firmware on the device to see if the older firmware version is what is causing issues with tethering.

Do note that activating any SIM card in the device will likely mean that you can now be tracked by an IMSI catcher, as the SIM will be registered in your name. When signing up for a Telnyx account, I had to give them my name and address to verify my account. This makes sense as they need to fight abuse, but just keep that in mind if you decide to active any service on the device. Unless you bought your Orbic directly from Verizon and they register the SIM with the sale of the device, if the IMSI of the device is captured, law enforcement (likely) wouldn't be able to tell who the device belongs to. If you have your phone on you though, its IMSI would have already been captured so this may be a moot point, but I did want to mention it. I'm also not an expert, so it's possible you may be tracked and identified no matter what.