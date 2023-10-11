---
layout: post
title:  "How I accidentally DoSed my own network"
date:   2023-07-28 04:13:13 +0000
categories: homelab
tags: [proxmox, networking, troubleshooting, arp, dhcp]
---

As the sole person tasked with configuring the network in the family, I was surprised when my brother one night told me that he was not able to access his banking website.

Upon hearing this I began to run the basic diagnostic steps and observed the following:

* The error code shown was ERR_ADDRESS_NOT_REACHABLE
* The site was unreachable despite changing browsers and restarting the operating system
* Pinging the IP address directly did not return any packets
* Other websites like google.com was completely accessible (!)
* Other devices could visit the affected site (!)
* Changing networks to a temporary hotspot resolved the issue (!)

The last three observations posed a bad "smell" as isolating for a variable usually appears to create a binary effect on the problem: Either the theoretical solution fixes the problem, or it does nothing. In this case, it appears that the root problem lies within the network. But how would the network be the problem when other devices in the same network can reach the site just fine? Shouldn't a network related problem affect all devices in the same network equally? Unless...

> ...isolating for a variable usually appears to create a binary effect on the problem: Either the theoretical solution fixes the problem, or it does nothing.

Unfortunately my naive belief that the problem was a simple fix(which in hindsight, actually was) led to false conclusions that the PC was completely at fault, despite no changes being done to PC for the past week before the issue was reported. After roughly an hour of hopeless troubleshooting and even changing the router's DNS configuration, I accessed the network logs to see if it might provide some helpful information. Here was what I found:

{% highlight bash %}
fakeuser@definitelynotrt:/tmp/home/root# cat /tmp/syslog.log | grep 'a6:2f'
Jul 11 20:23:44 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:23:50 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:23:59 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:24:13 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:24:29 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:24:37 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:29:36 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 
Jul 11 20:29:36 dnsmasq-dhcp[23025]: DHCPOFFER(br0) 192.168.1.153 a6:2f:be:d4:65:46 
Jul 11 20:29:36 dnsmasq-dhcp[23025]: DHCPREQUEST(br0) 192.168.1.153 a6:2f:be:d4:65:46 
Jul 11 20:29:36 dnsmasq-dhcp[23025]: DHCPACK(br0) 192.168.1.153 a6:2f:be:d4:65:46 nginx
Jul 11 20:29:36 dnsmasq-dhcp[23025]: DHCPDECLINE(br0) 192.168.1.153 a6:2f:be:d4:65:46 
Jul 11 20:29:46 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:29:53 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:30:07 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:30:26 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
Jul 11 20:30:37 dnsmasq-dhcp[23025]: DHCPDISCOVER(br0) a6:2f:be:d4:65:46 no address available
{% endhighlight %}

Ok, cool, now we're getting somewhere. Based off the logs provided, it appears that some devices are not receiving a IP from the DHCP server. Should be no problem, let's assign a DHCP reservation to the affected PC and get this over with.

> Based off the logs provided, it appears that some devices are not receiving a IP from the DHCP server. 

Except that the assigned PC still didn't get an IP reservation and the PC still couldn't access the site.

Ok, maybe I was just unlucky. Maybe I accidentally typed an static ip that was already taken by another client. This time, let's try an static ip address of 192.168.1.88.

No dice. Let's try 192.168.1.89.

Nope. Now 192.168.1.90.

Nada. One more time for 192.168.1.91...

Now at this point I'm sweating bullets because something is clearly wrong, as last time I checked our DHCP reservation range was from 192.168.1.2 to 192.168.1.255, and we didn't have 254 unique devices in the house. Out of curiosity to see which IP addresses are open, I ran a blind arp(conversion hash table for IP to MAC addresses) scan:

{% highlight bash %}
cs@CS:~$ sudo arp-scan --localnet
Interface: wlp1s0, type: EN10MB, MAC: f0:d5:bf:24:88:3f, IPv4: 192.168.1.210
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.1.1	1c:b7:2c:c6:c7:f0	(Unknown)
192.168.1.2	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.3	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.4	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.5	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.6	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.7	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.8	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.9	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.10	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.11	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.12	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.13	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.15	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.16	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.17	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.18	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.19	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.20	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.21	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.22	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.23	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.24	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.25	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.26	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.27	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.28	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.29	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.30	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.31	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.32	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.33	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.34	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.35	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.36	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.37	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.38	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.39	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.40	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.41	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.42	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.43	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.44	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.45	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.46	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.47	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.48	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.49	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.50	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.51	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.52	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.53	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.54	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.55	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.56	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.57	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.59	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.60	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.61	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.62	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.63	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.64	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.65	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.67	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.68	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.69	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.72	94:de:80:62:60:9a	(Unknown)
192.168.1.70	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.71	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.73	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.74	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.75	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.76	44:8a:5b:d0:85:d1	(Unknown)
192.168.1.77	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.78	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.79	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.81	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.84	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.85	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.83	e2:7e:1c:8d:ce:d6	(Unknown: locally administered)
192.168.1.86	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.87	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.88	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.89	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.90	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.91	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.92	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.93	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.95	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.96	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.97	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.98	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.100	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.101	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.94	1c:b7:2c:c7:61:e8	(Unknown)
192.168.1.102	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.103	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.104	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.105	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.106	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.107	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.108	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.109	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.110	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.111	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.112	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.113	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.115	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.116	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.117	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.118	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.119	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.121	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.122	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.124	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.125	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.126	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.127	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.128	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.129	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.130	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.131	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.123	3c:a3:08:3c:d7:05	(Unknown)
192.168.1.132	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.133	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.135	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.137	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.138	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.139	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.140	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.141	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.143	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.144	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.145	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.146	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.147	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.148	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.149	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.150	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.151	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.142	e4:c3:2a:a0:31:08	(Unknown)
192.168.1.152	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.153	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.154	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.155	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.156	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.157	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.158	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.159	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.160	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.161	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.162	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.163	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.164	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.166	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.167	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.168	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.169	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.170	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.171	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.172	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.173	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.165	52:44:b5:22:8d:1c	(Unknown: locally administered)
192.168.1.174	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.175	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.176	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.177	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.178	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.180	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.181	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.182	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.183	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.185	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.186	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.187	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.188	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.189	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.190	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.191	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.192	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.193	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.194	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.196	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.197	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.198	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.199	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.200	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.202	18:fd:cb:b0:43:e4	(Unknown)
192.168.1.195	f4:6d:04:65:4c:85	(Unknown)
192.168.1.201	18:fd:cb:b0:43:92	(Unknown)
192.168.1.204	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.205	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.206	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.207	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.208	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.209	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.212	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.203	a0:62:fb:81:38:bc	(Unknown)
192.168.1.213	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.214	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.215	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.216	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.217	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.218	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.219	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.120	50:fd:d5:97:08:9b	(Unknown)
192.168.1.136	42:00:7f:58:dc:5e	(Unknown: locally administered)
192.168.1.179	5c:ba:ef:e4:40:27	(Unknown)
192.168.1.99	c4:29:96:09:d4:a0	(Unknown)
192.168.1.221	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.222	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.223	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.224	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.225	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.226	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.227	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.228	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.229	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.230	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.231	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.234	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.235	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.236	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.237	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.238	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.239	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.240	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.241	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.242	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.243	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.244	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.245	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.246	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.247	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.249	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.250	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.252	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.253	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.254	a6:2f:be:d4:65:46	(Unknown: locally administered)
192.168.1.251	ec:b5:fa:9b:d7:9c	(Unknown)
192.168.1.114	84:72:07:cc:fc:45	(Unknown)
192.168.1.99	c4:29:96:09:d4:a0	(Unknown) (DUP: 2)
192.168.1.66	ac:63:be:9e:dd:bc	(Unknown)
192.168.1.232	86:94:e4:a4:2c:85	(Unknown: locally administered)
192.168.1.58	46:03:af:d9:26:1c	(Unknown: locally administered)
{% endhighlight %}

Aha! I knew it! Some jackass sneaked into my network and started a [DHCP Starvation Attack](https://www.cbtnuggets.com/blog/technology/networking/what-is-a-dhcp-starvation-attack)! Let's get on the router and ban this fool *"a6:2f:be:d4:65:46"* so we can at least open some IP addresses. 

Wait a second...*"a6:2f:be:d4:65:46"* looks familiar...wasn't that a machine I had to manually reserve a static IP for? Hold up...that's my own nginx virtual machine. I'm the one hogging all the IPs. That "jackass" was me.

> That "jackass" was me.

I immediately checked my virtual machines, and sure enough, most of them were running but unable to read/write due to a separate problem with a faulty SSD I had ignored from the previous night...

What happened was that somewhere along the line of my SSD disconnecting and the nginx VM attempting to access/write a file, the I/O failure somehow prompted nginx to request and hold every DHCP address available in the subnet masks in a loop, which explains the DHCP requests, ARP table, and why some of the devices that were already assigned an ip address showed no symptoms. 

After taking 5 seconds to turn off my virtualization server, the IP addresses cleared up and the PC was finally able to access chase.com:

{% highlight bash %}
cs@CS:~$ sudo arp-scan --localnet
Interface: wlp1s0, type: EN10MB, MAC: f0:d5:bf:24:88:3f, IPv4: 192.168.1.210
WARNING: Cannot open MAC/Vendor file ieee-oui.txt: Permission denied
WARNING: Cannot open MAC/Vendor file mac-vendor.txt: Permission denied
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.1.1	1c:b7:2c:c6:c7:f0	(Unknown)
192.168.1.66	ac:63:be:9e:dd:bc	(Unknown)
192.168.1.76	44:8a:5b:d0:85:d1	(Unknown)
192.168.1.82	68:b9:c2:13:04:25	(Unknown)
192.168.1.94	1c:b7:2c:c7:61:e8	(Unknown)
192.168.1.123	3c:a3:08:3c:d7:05	(Unknown)
192.168.1.99	c4:29:96:09:d4:a0	(Unknown)
192.168.1.195	f4:6d:04:65:4c:85	(Unknown)
192.168.1.201	18:fd:cb:b0:43:92	(Unknown)
192.168.1.202	18:fd:cb:b0:43:e4	(Unknown)
192.168.1.203	a0:62:fb:81:38:bc	(Unknown)
192.168.1.251	ec:b5:fa:9b:d7:9c	(Unknown)
192.168.1.91	38:ba:f8:43:ee:19	(Unknown)
192.168.1.179	5c:ba:ef:e4:40:27	(Unknown)
192.168.1.136	42:00:7f:58:dc:5e	(Unknown: locally administered)
192.168.1.14	46:88:38:f1:3d:59	(Unknown: locally administered)
192.168.1.120	50:fd:d5:97:08:9b	(Unknown)
192.168.1.232	86:94:e4:a4:2c:85	(Unknown: locally administered)
{% endhighlight %}

And that's the story on how I accidentally banged my head against a wall for 2 hours. 

