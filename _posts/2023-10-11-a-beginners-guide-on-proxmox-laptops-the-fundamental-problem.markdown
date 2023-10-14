---
layout: post
title:  "A Beginner's Guide on Proxmox Laptops: The Fundamental Problem"
date:   2023-10-11 20:39:13 +0000
categories: homelab
tags: [proxmox, networking, guide, mac, bridge, vm]
feature_image: laptop_server.jpg
feature_caption: "A frankenstein laptop server"
---
{% include image_finder.html %}
## Incentives

If you are like me, you may have had a couple of used laptops around the house and wondered if it was ever possible to extend the usability of these older devices by installing Proxmox and spinning some virtual machines. In fact, re-purposing my laptop as a Proxmox instance to setup local development servers was a keystone for my earlier development days, allowing convenient access to my coding tools without purchasing a new dedicated machine.

<!--more-->

Regardless of the reason, I would like to warn you that due to technical problems, using Proxmox on a wireless laptop can be a tricky endeavor to properly execute, especially for beginners with no prior knowledge of networking. In those cases, it would be best to simply follow the official Proxmox guide with the laptop connected via Ethernet, forever.

For those without a faint of heart, you may continue reading to create a Proxmox laptop that can essentially be brought with you anywhere.

## The Fundamental Problem

Firstly, an overall picture of why Ethernet is supported on Proxmox but not wireless. The fundamental problem is that how the wired network management between hosts and VMs were designed prevents a wireless computer from communicating similarly as a byproduct. 

That's a lot to digest. Let's break it down.

### Digestion

For a VM to be able to visits a page on the Internet, the VM first must communicate to the bare-metal host to access the web for them as a middleman. If you think about it, both the VM and host share the same physical resources, but since these physical resources are owned by the host, it seems reasonable for the VM to always ask the host for permission on these resources.

But since communication from VM to Host are an essential requirement for the VM to function, some intelligent and formal process must be created to efficiently facilitate this connection. 

This formal process by which a host and a set of VMs talk over a network in Proxmox is called bridging. In a bridging scenario, there exists two locations by which the only way to transverse between them is through one central point called the bridge. In this example, the host bridges network traffic between the VMs and the rest of the Internet, as all VMs must talk to the host before talking to the Internet, and the Internet must talk to the host before being able to talk to the VMs. 

![Proxmox Bridge]({{IMG}}ProxmoxBridge.svg)
*An simplified example of a Proxmox bridge*

Each VM are given an unique identifier called a MAC address. These addresses are particularly useful for distinguishing network traffic from one another. Specifically, network traffic will always contain a source and destination MAC addresses to know where the data came from and where it should go. Think of the source and destination as a return and recipient address of a letter, respectively. 

Under a regular Ethernet-wired Proxmox bridge, a VM called "A" will send data with its corresponding MAC address stamped on it. Since the host knows this data belongs to "A", it can intelligently use this MAC address information to forward the data to the router(Internet), similarly to how a deliveryman drives to a local UPS center to drop off a particular item before it gets shipped to its next intermediary destination.  

![Packet Filter](https://media.geeksforgeeks.org/wp-content/uploads/1111-6.png)
*An example of a packet filter. In our case, the wireless router is examining data pieces(packets) to see if they came from a trusted, authenticated source. Since our VM has never authenticated before, any data sent will go straight to the dustbin. Courtesy of [Geeks for Geeks](https://www.geeksforgeeks.org/types-of-firewall-and-possible-attacks/#)*

Under a wireless Proxmox bridge, however, the router is extremely picky for which source addresses are allowed to be forwarded. As far as the router is concerned, it is aware of VM "A"'s forwarded source MAC address, but this particular MAC has never been in the "approved list", so to speak, as all devices undergoing the authentication procedures of a secure, password-protected network are. This is commonly why Proxmox laptops tends to be able to access the Internet through the host but the VMs themselves can't, since the host's MAC address were properly authenticated but the VMs's MAC addresses were not.

## But Why

This supposed restriction is interestingly a designed security feature for most wireless access points. Specifically, the intention behind blocking non-authenticated source MAC addresses was to gatekeep hackers from maniuplating the network without a password. Typically, a wired connected device will be  located in a place with some form of physical security - a security guard, surveilence cameras, or even the door to your house - so the router has less risk and therefore more inherent trust in the device. 

But wireless data is quite litteraly in the air and can be intercepted by anyone at anytime. I mean think about it: an malicious actor can theoretically break into your network just by chilling in the car parked outside your house with a laptop and you would be none the wiser. 

## What now
So now the router's a bouncer and it's not letting any of the non-VIP VMs in. How do we get around it? In the next article, we will discuss ARP and how ARP can be used to help us accomplish our goals. Be sure to let the information slowly digest in your brain, because having a solid understanding of why wireless Proxmox can be difficult will help you in the next steps.

### References:

* [Why can't we easily bridge over a wireless NIC](https://unix.stackexchange.com/questions/444960/why-cant-we-easily-bridge-over-a-wireless-nic)

* [Proxmox WLAN Wiki](https://pve.proxmox.com/wiki/WLAN)

* [VirtualBox bridged network and WLAN](http://nullroute.lt/~grawity/journal-2011.html#post:20110826)