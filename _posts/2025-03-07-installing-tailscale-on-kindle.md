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

About a year ago I jailbroke my Kindle after coming across a YouTube video where someone was showing off the Winterbreak jailbreak. Initially, I only wanted to jailbreak my Kindle because it seemed like fun. I didn't particularly care to use KOReader, as I didn't really have any complaints with the stock reader. I had been using the "Send to Kindle" feature of Calibre-Web Automated to send my DRM-free books I was purchasing from Kobo over to my Kindle, so I was still able to enjoy all of my DRM-free books on the Kindle. After jailbreaking my Kindle and using KOReader for a while, I really started to like the customization that it offered. After messing with it even more, the sync feature for Calibre-Web Automated was much easier than having to use the "Send to Kindle" feature, which often took several minutes for the book to show up on my Kindle. I also wasn't the biggest fan of Jeff Bezos knowing what I was reading, so this was a nice way to get around that as well.

Since I jailbroke my Kindle, I have helped several others jailbreak their Kindle's as well (I'm up to 4, including my own), and now we are all spreading the gospel of owning your own books far and wide, and we have a few more people who seem to be interested in jailbreaking their device as well.

While talking my friend through jailbreaking his wife's Kindle, he mentioned it would be neat if he could run Tailscale on his Kindle so he could access his books from anywhere. That lead me down the rabbit hole of putting Tailscale on yet another device that has no business running Tailscale.

> I personally have a Kindle PW5 (Paperwhite 5) running firmware version 5.17.1.0.3, so this guide will be centered around that device and firmware version. This SHOULD work for all newer firmware versions as well. If something here doesn't work for you, please leave a comment down below so I can update this guide.
{: .prompt-info }

## Installing USBNetworking

I was very annoyed when trying to install USBNetworking, and that is the whole reason I decided to write my own post about how to do this. There are a few other guides out there, as well as a readme that is included with the official USBNetworking Hack distributed on [**MobileReads**](https://www.mobileread.com/forums/showthread.php?t=225030), but I couldn't get that version working, and every single site I came across that had install instructions for this was outdated and incorrect.

Today, I am going to be loosely following the guide from [**mitanshu7**](https://github.com/mitanshu7/tailscale_kual?tab=readme-ov-file) that was linked from the official [**Tailscale blog post**](https://tailscale.com/blog/tailscale-jailbroken-kindle) on how to run Tailscale on your Kindle. I chose to use the version without Taildrop because I personally don't need Taildrop since I have a Calibre-Web Automated server that I will use for transferring books.

The first thing you will need to do is install USBNetworking, and as mentioned earlier, this was a huge frustration for me. I followed the instructions in the readme that came packaged with with USBNetworking Hack, but I couldn't get it to install! This is when I started searching around, and came across countless articles all saying one of two (incorrect) things. They either said to:
1. Copy the `Update_usbnet_*.bin` file over to the `mrpackage` folder on the Kindle and install it from KUAL; or
2. Copy the `Update_usbnet*.bin` package to the root of the Kindle, and do a firmware update.

Unfortunately, neither of these worked for me. After doing some more digging, I found out that the issue is that starting in Kindle firmware version 5.16.3, the architecture was moved to armhf, and the packages provided from MobileReads are only compiled for the old architecture prior to 5.16.3. To install USBNetworking now, you need to install [**USBNetLite**](https://github.com/notmarek/kindle-usbnetlite), which is compiled for armhf.

Installing USBNetLite is straightforward. Download the latest release release from the above GitHub page (in my case, I downloaded the one for `11thgenplus`), then copy the file to the `mrpackage` folder on your Kindle. Launch KUAL, and go to `Helper`, `Install MR Packages`, and wait for the install to finish. You may get a popup warning you that an application may not be able to launch. That's normal, just accept it and move on. After your Kindle restarts, you should have a new option in KUAL for USBNetLite.

### Configuring SSH

Since you will need to keep ssh enabled for at least the USB network in order for Tailscale to work, I might as well copy over my public key and disable password authentication. I'm going to go ahead and tap `Toggle USBNetwork`, then tap `USBNetwork Status`. It should say `USBNetwork: enabled (usbnet, sshd up)`. If it doesn't show this as enabled, give it a few seconds and refresh the status. It usually takes a few seconds to start, so be patient. Now, I will ssh using the default username (root) and default password (kindle) and copy my public key.

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

Now the fun part! From [**mitanshu7's**](https://github.com/mitanshu7/tailscale_kual?tab=readme-ov-file) repo, download the whole thing as a zip (Code > Download ZIP), then extract it. We need to make a few changes before we copy anything over to our Kindle. First of all, go over to [**Tailscale**](https://login.tailscale.com/admin/settings/keys) and generate an auth key. Note that the "Expiration" is only for when the single-use auth key will expire, NOT for when the client key will expire. Paste your generated key into the file `auth.key` file, which is at `tailscale/bin`.

Now, copy the entire `tailscale` folder into to your Kindle's `extensions` folder. After you copy the folder over, go back to KUAL and you should see a new entry for Tailscale. The first thing we will need to do is install Tailscale, so click `Install / Update Binaries` to automatically have your Kindle download the latest Tailscale version. Keep in mind that the Kindle is not fast, so this may take a minute or two to complete. After you see the `Install complete` message, you should be good to go. Make sure that ssh is running, then from the Tailscale menu, go to `Start Tailscaled`, then select `Kernel TUN (if supported)`. I am opting to use kernel networking mode, as the userspace networking modes do not allow the device itself to access other devices on your Tailnet, which is a requirement if I want to use this with Calibre-Web Automated as I mentioned earlier. After that, you just need to go back to the previous page and select `Start Tailscale`, and you should see a message pop up on screen to inform you that you are connecting. Now, you should be able to see your device in your device list! From here, you can disable key expirey for the node so you don't have to update the key when it expires.

One weird thing I came across was that my `Start Tailscale` button wasn't working for some reason. I would tap it and nothing would happen. I checked the `start_tailscale.sh` file on my Kindle and it was blank! I had to plug my Kindle back in and copy the file over again, then that fixed the issue.

If you are using KOReader, when you launch KOReader, make sure to NOT launch it with the `no framework` option, as this will kill the Tailscale processes, which is a real bummer.