---
title: Installing Tailscale on a jailbroken Kindle PW5 
description: Installing tailscale and all associated components on a jailbroken Kindle
author: Smith8154
date: 2026-03-07 23:00:00
categories: [Personal]
tags: [Tailscale, Kindle, Jailbreaking]
pin: false
math: false
mermaid: true
---

## Tailscale on a Kindle?

I have started something of a revolution among my friends. It started when I jailbroke my Kindle about a year ago, and from there it has spread like wildfire. I have jailbroken my wife's kindle and two other Kindle's for my friends. Since then, my friends have been spreading the gospel of owning your own books far and wide, and there may be some others interested in jailbreaking their Kindles as well.

While talking my friend through jailbreaking his wife's Kindle, he mentioned it would be neat if he could run Tailscale on his Kindle so he could access his books from anywhere. That lead me down the rabbit hole of putting Tailscale on yet another device that has no business running Tailscale.

I personally have a Kindle PW5 (Paperwhite 5), so this guide will be centered around that device.

## Installing USBNetworking

I was very annoyed when trying to install USBNetworking, and that is the whole reason I decided to write my own post about how to do this. There are a few other guides out there, as well as a readme that is included with the official USBNetworking Hack distributed on [**MobileReads**](https://www.mobileread.com/forums/showthread.php?t=225030), but I personally found the install process to be confusing, and every single site I came across that had install instructions for this was outdated and incorrect.

Today, I am going to be loosely following the guide from [**mitanshu7**](https://github.com/mitanshu7/tailscale_kual?tab=readme-ov-file) that was linked from the official [**Tailscale blog post**](https://tailscale.com/blog/tailscale-jailbroken-kindle) on how to run Tailscale on your Kindle. I chose to use the version without Taildrop because I personally don't need Taildrop since I have a Calibre-Web Automated server that I will use for transferring books.

The first thing you will need to do is install USBNetworking, and as mentioned earlier, this was a huge frustration for me. I followed the instructions in the readme that came packaged with with USBNetworking Hack, but I couldn't get it to install! This is when I started searching around, and came across countless articles all saying one of two (incorrect) things. They either said to:
1. Copy the `Update_usbnet_*.bin` file over to the `mrpackage` folder on the Kindle and install it from KUAL; or
2. Copy the `Update_usbnet*.bin` package to the root of the Kindle, and do a firmware update.

Neither of these worked. Maybe they worked at one point in time, but they work no longer and the posts haven't been updated to reflect the changes. After doing some more digging, I found out that the issue is that starting in Kindle firmware version 5.16.3, the architecture was moved to armhf, and the packages provided from MobileReads are only compiled for the old architecture prior to 5.16.3. To install USBNetworking now, you need to install [**USBNetLite**](https://github.com/notmarek/kindle-usbnetlite), which is compiled for armhf.

Installing USBNetLite is straightforward. Download the latest release release from the above GitHub page (in my case, I downloaded the one for `11thgenplus`), then copy the file to the `mrpackage` folder on your Kindle. Launch KUAL, and go to Helper, Install MR Packages, and wait for the install to finish. You may get a popup warning you that an application may not be able to launch. That's normal, just accept it and move on. After your Kindle restarts, you should have a new option in KUAL for USBNetLite.

### Configuring SSH

I will probably end up keeping ssh disabled, but while I'm here, I might as well copy over my public key and disable password authentication. I'm going to go ahead and tap `Toggle USBNetwork`, then tap `USBNetwork Status`. It should say `USBNetwork: enabled (usbnet, sshd up)`. Now, I will ssh using the default username (root) and default password (kindle) and copy my public key.

> If you do not plan on using private key authentication, I HIGHLY recommend blocking ssh over WiFi for security reasons. If you keep ssh enabled on wifi with the default credentials, anyone on the network could gain access to your Kindle, and therefore other devices in your Tailnet.
{: .prompt-warning }

```shell
vi /mnt/us/usbnetlite/etc/dropbear/authorized_keys
# Paste private key
```

Now to disable password authentication, we just need to update one line in the config file.

```shell
sed -i 's/ALLOW_PASSWORD_LOGIN="true"/ALLOW_PASSWORD_LOGIN="false"/' /mnt/us/usbnetlite/etc/config
```

## Installing Tailscale

Now the fun part! From [**mitanshu7's**](https://github.com/mitanshu7/tailscale_kual?tab=readme-ov-file) repo, download the whole thing as a zip (Code > Download ZIP), then extract it. We need to make a few changes before we copy anything over to our Kindle. First of all, go over to [**Tailscale**](https://login.tailscale.com/admin/settings/keys) and generate an auth key. Paste your generated key into the file `auth.key` file, which is at `tailscale/bin`.

Now, copy the entire `tailscale` folder over to your Kindle. It needs to go in the `extensions` folder. After you copy the folder over, go back to KUAL and you should see a new entry for Tailscale. The first thing we will need to do is install Tailscale, so click `Install / Update Binaries` to automatically have your Kindle download the latest Tailscale version. Keep in mind that the Kindle is not fast, so this may take a minute or two to complete. After you see the `Install complete` message, you should be good to go. Make sure that ssh is running, then from the Tailscale menu, go to `Start Tailscaled`, then select `Kernel TUN (if supported)`. I am opting to use kernel networking mode, as the userspace networking modes do not allow the device itself to access other devices on your Tailnet, which is a requirement if I want to use this with CWA as I mentioned earlier. After that, you just need to go back to the previous page and select `Start Tailscale`, and you should see a message pop up on screen to inform you that you are connecting. Now, you should be able to see your device in your device list!

One weird thing I came across was that my `Start Tailscale` button wasn't working for some reason. I would tap it and nothing would happen. I checked the `start_tailscale.sh` file on my Kindle and it was blank! I had to plug my Kindle back in and copy the file over again, then that fixed the issue.