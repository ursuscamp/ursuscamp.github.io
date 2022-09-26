---
layout: post
title:  "Home Server 2: Installing Ubuntu"
date:   2022-09-25 22:46:45 -0400
categories: bitcoin server
---
A computer is no good without software, so it's time to get the operating system up and running. We will be installing Ubuntu Server 22.04. Ubuntu Server is a simple and quick install with a non-graphical O.S. It comes with OpenSSH available by default, and it is widely supported.

Download [balenaEtcher](https://www.balena.io/etcher/) which I will use to write the boot drive, and [Ubuntu Server 22.04](https://ubuntu.com/download/server) ISO. Insert the USB drive we will use to install, then start Etcher and select the ISO and drive, then follow the prompts to burn the image to the USB.

After that is finished, reboot the computer and go into the BIOS menu. This is vendor specific and you will need to look it up on the internet if you do not already know it. On my PC, as soon as I start the computer, I tap F2 until the BIOS comes up. After doing so, you will need to edit the boot order so that it looks at the USB drive first. Also, on my PC, I had to turn on UEFI secure boot, because after installation my computer refused to boot the Ubuntu install without it.

When the USB drive boots, you will be greeted by this screen:

![first screen](/assets/2022-09-25/001.png)

Select `Try or Install Ubuntu Server` to proceed to the next stage, where you will select your language and keyboard layout.

![select language](/assets/2022-09-25/002.png)
![select layout](/assets/2022-09-25/003.png)

Install Ubuntu Server (not the minimized version).

![install type](/assets/2022-09-25/004.png)

You will be prompted to select your network interface. By default here, you will see your ethernet device, which should be plugged in directly to your home router or modem. Please take note of your local IP address here, this is how you will connect to your machine over the your home network.

![network interface](/assets/2022-09-25/005.png)

If you have a proxy, configure it now. I do not, so I left it blank.

![proxy](/assets/2022-09-25/006.png)

Leave the mirror address as the default.

![mirror](/assets/2022-09-25/007.png)

Create your hard disk where you will install, but also de-select LVM as it is not needed and will add unnecesasry overhead.

![disable LVM](/assets/2022-09-25/008.png)

Let Ubuntu handle formatting your drive partitions with the default options. You should see a lot more space here, as these screenshots were taken in a VM with limited space.

![setup partitions](/assets/2022-09-25/009.png)

Setup your default user and password. This is how you will identify yourself to your PC when you log in over the network.

![user setup](/assets/2022-09-25/010.png)

Make sure to select to install OpenSSH server, or you will not be able to log into your server over the network.

![install openssh](/assets/2022-09-25/011.png)

There's no need to modify any of the default installed software. Skip right on past this screen.

![installed software](/assets/2022-09-25/012.png)

Finally the isntall will begin. It should only take a few minutes.

![install commenced](/assets/2022-09-25/013.png)

When you're finished, you will be prompted to remove the install medium (USB drive) and reboot the machine. Once you do, you will hopefully be greeted with the login page for the user you setup!

Links:

https://ubuntu.com/tutorials/install-ubuntu-server#1-overview