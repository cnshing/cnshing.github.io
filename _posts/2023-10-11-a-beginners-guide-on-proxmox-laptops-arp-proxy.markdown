---
layout: post
title:  "A Beginner's Guide on Proxmox Laptops: ARP Proxy"
date:   2023-10-16 12:11:37 +0000
categories: homelab
tags: [proxmox, networking, guide, arp, bridge, vm]
feature_image: ARP_Spiderman_Meme.png
feature_caption: "An ARP conflict where two MAC addresses claim to be the same IP. Courtesy of [McWooky](https://www.reddit.com/r/MemeRestoration/comments/eq1jwh/i_restored_in_hd_4k_the_original_spiderman/)."
---
{% include image_finder.html %}

In the last post, we mentioned a particular issue when it comes to wirelessly connecting our virtual machines to the Internet under a Proxmox host. Specifically, our previous method of directly informing the router of each VM's MAC address is automatically rejected as they have not made any attempt to properly authenticate with the router, even if the Proxmox device hosting all the VMs did.

Now, the network can be a complicated system of interconnected physical links and data abstraction[^1], all of which abide by a set of known and predictable rules. In fact, the very problem of our Proxmox Laptop is itself a rule to prevent unauthorized access via the physical radio wireless waves of a network. This complexity in networking allows a variety of methods to use a particular rule from a particular part of the whole network ecosystem to accomplish the same goal.

The first method is to hide the VM's presence by masquerading their network traffic as the host's own. From the router's perspective, all communication appears to originate from the Proxmox host, but internally calculations are done to intelligently redirect all the traffic to the appropriate VMs. This "redirection" or translation of network traffic can be achieved by maniuplating the source MAC addresses directly, or through the IP addresses, both of which are separate, but related parts of a network used for identification purposes. See [etables](https://wiki.debian.org/BridgeNetworkConnections#Bridging_with_a_wireless_NIC) and [iptables](https://pve.proxmox.com/wiki/Network_Configuration#sysadmin_network_masquerading) for more information.

The second method is to make the router believe all the VMs are located in an entirely separate network. The logic is that if a router rejects unauthenticated traffic inside its own network, then just tell the router the traffic is from a different network, thereby allowing this specific authentication check to be skipped. Our separate network approach is considered to be "transparent" because nothing is being purposely hidden or manipulated - all VMs are transparently available for the router to see. Since every network device was practically designed to pass along someone else's data like a perfect game of telephone, the router will treat the Proxmox host as simply another router to move traffic from the home network to the VM network. 

Two known approaches can be done to separate the networks and enable communications, one of which is [Wireless Distribution System](https://www.cisco.com/assets/sol/sb/AP541N_Emulators/AP541N_Emulator_v1.9.2/help_WDS_Bridge.htm) or WDS. However, because WDS can sometimes require you to essentially flip your network upside down to get it working, our method of choice is an ARP proxy for the following reasons:

* Address Resolution Protocol is an extremely important and fundamental concept for networking and applying these ideas in a real and practical environment can be incredibly helpful in reinforcing these concepts.
* Less complexity to deal with as no manipulation of MAC addresses or IP addresses is required.
* A bit more user-friendly due to more supporting [documentation](https://wiki.debian.org/BridgeNetworkConnectionsProxyArp)
* I personally used ARP to get my Proxmox Laptop running, and I'm not too sure about the other methods, especially WDS.

Before we go in-depth on the ARP Proxy, a brief explanation of what ARP is and why it matters. When it comes to sending a file, there are two distinct paths the file can take: a path contained in one local network, or a path moving between multiple networks. Similarly to driving a car to your grocery store versus boarding a plane at the airport, the physical limitations on the file's path can induce a lot of variability in the performance and reliability of how a data connection operates. Think about AirDropping a file between two iPhones versus sending the same file from London to Hong Kong. The file would only need to communicate just over a couple of inches with blazing fast speeds and latency, while the other must quite literally travel across oceans to reach the destination. Now imagine you are tasked with sending this file and potentially many others en masse to various destinations. The conclusion is most likely that it would be incredibly beneficial to have a separate transportation procedure for files just within the local network and for files meant to travel outside the network.

In relation to ARP, identifying other hosts within the same local network uses MAC addresses, while IP addresses were used for identification across networks. One can infer a host would then need both a MAC address to not only communicate between local devices within the same network, but also an IP address to communicate outside the network(to access the Internet). 

However, what if someone had only the IP address of a host, but not the MAC address? How would they communicate then?

The Address Resolution Protocol was designed to answer that question by bridging the gap between MAC and IP addresses, and the protocol is actually quite simple. Specifically, every device adhering to ARP will have to maintain their own personal list of associated MAC and IP addresses they had seen, like meeting a bunch of strangers at a party and taking note of their name. When a host needs to know a particular device's MAC, the host will shout across the entrie network, "Nice to meet you! I am host H with MAC XYZ and IP 123. Is there anyone with an IP of 456? I need their MAC!", allowing everyone else hearing H's message to take note of the pair of MAC address and IP along the way. If the host is lucky, someone might respond in kind with the right MAC address to H, allowing H to successfully communicate.

![ARP Response Request](https://www.practicalnetworking.net/wp-content/uploads/2017/01/traditional-arp-process.gif)
*A high level illustration on ARP. Courtesy of [Practical Networking](https://www.practicalnetworking.net/series/arp/traditional-arp/).*

The role ARP will play in our Proxmox laptop is to give the router the VM's MAC and IP addresses so that all the information is available to successfully create a connection between the VM and the router, and therefore the Internet. The implementation of this technique is called an "proxy" because the Proxmox laptop is answering any MAC requests on behalf of the VMs and the router: The VMs and Wireless router never directly receive their ARP information from each other like how two attorneys may prevent their respective clients from directly meeting and potentially self-incriminate themselves in a divorce case. 

![Proxmox ARP Proxy]({{IMG}}Proxmox_ARP_Proxy.svg)
*A Proxmox host wirelessly bridging its VMs and the router for Internet access. The VM and Home network IPs were made different to show they can be treated as two separate networks with Proxmox sitting in the middle to route data between them. The wireless router will not think twice about this new logically connected network because the Proxmox host responsible for routing the network is properly authenticated.*

I hope that you were able to learn a lot about a few tidbits on how a network works, what Address Resolution Protocol is, and how we can use ARP to theoretically create a working connection between our VMs and the rest of the Internet. In the next article, we will start putting the theory into practice by showing the implementation steps of an ARP Proxy and a couple of practical tips on installing and navigating Proxmox through a laptop.

[^1]: The act of transforming a typically monotonous data to an equivalent, but smaller or reduced form representing only the essential characteristics. Our example refers to deciphering a extremely repetitive amount of binary 1s and 0s on a physical link into human readable language.
