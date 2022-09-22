---
layout: post
title:  "Home Server 1: Bitcoin Node Hardware Setup"
date:   2022-09-20 21:31:23 -0400
categories: bitcoin server
---
I decided it was finally time to setup a Bitcoin node of my very own. I debated a long time about
getting a Raspberry Pi or buying a NUC-like device. Finally, I decided to repurpose an old laptop
I have, and I think that I made the right decision.

Initially I worried that the fan noise would annoy me or my wife (our router is in our bedroom).
However, so far, the noise is minimal to non-existent. I suppose running a Bitcoin node is not
a particularly taxing task. As I add more services to the device, I will see how it continues
to fare.

## The Machine

The laptop I'm using is an [Acer Aspire E5-575G](https://www.amazon.com/gp/product/B01LD4MGY4) that I
ordered from Amazon 5 years ago. It's CPU is an Intel Core i5-7200U CPU @ 2.50GHz and it has 8GB of
DDR4 RAM.

It has a 256GB SSD drive, which I replaced with a [Western Digital 1TB SSD drive](https://www.walmart.com/ip/WD-Blue-1TB-SN570-NVMe-SSD-WDBB9E0010BNC-WRSN/465417170) which I purhased from Walmart. One terabyte is already
getting low for a Bitcoin full node + extra services (such as an Electrum server) so I will
no doubt need to update that in the future.

I also opted to remove the battery. It probably wasn't necessary, but the idea of leaving a laptop on,
day and night, plugged in with a battery made me nervous.

Here are some useful tutorials:

* [Battery removal](https://www.youtube.com/watch?v=qMKzefxhhqY)
* [SSD replacement](https://www.youtube.com/watch?v=hIcqAA0l6DU)

## Home Internet

When it comes to running a Bitcoin node, network connection is important. I have a gigabit
fiber connection with AT&T, and my router is a [TP-Link Archer AX73](https://www.amazon.com/gp/product/B08TH4D3QV).
This router is a beast for home internet use. I know you might think you need a $500 mega-low latency
gaming router, but you don't. However, if you're using a default, ISP-provided router you **definitely**
need to upgrade.

I used the default router for years and eventually I had about 15 devices on it, between computers,
tablets, phones, smart switches, etc. AT&T only rated it for 10 devices and it showed. The internet
connection would die constantly, videos would buffer, on and on. After updgrading to this router,
we have had zero issues. There are now 20 devices connected and this thing still screams.

Initially I had this computer connected via WiFi so I could place it somewhere else in the house.
As I mentioned previously, I was worried about fan noise. It did work, actually. The Bitcoin node
fully synced in a very short time (maybe eight hours). However, I noticed lag while SSH'ing
into the machine. Connecting via ethernet directly to the router resolved that issue. It's not
absolutely necessary to use ethernet, but I would suggest it.

## Placement

It is currently tucked nearly underneath our dresser. I put a piece of electrical tape over the blue
LED indicator and it is upside down for airlow.

![node placement](/assets/2022-02-20/IMG_1954.jpeg)