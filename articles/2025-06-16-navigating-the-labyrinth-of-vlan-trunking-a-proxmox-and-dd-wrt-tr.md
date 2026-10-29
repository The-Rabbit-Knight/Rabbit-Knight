---
title: "Navigating the Labyrinth of VLAN Trunking: A Proxmox and DD-WRT Troubleshooting Saga"
date: 2025-06-16
medium_url: https://medium.com/@rabbit_knight/navigating-the-labyrinth-of-vlan-trunking-a-proxmox-and-dd-wrt-troubleshooting-saga-423afa66d9dc
---

> **Original Published Article on Medium:** [Navigating the Labyrinth of VLAN Trunking: A Proxmox and DD-WRT Troubleshooting Saga](https://medium.com/@rabbit_knight/navigating-the-labyrinth-of-vlan-trunking-a-proxmox-and-dd-wrt-troubleshooting-saga-423afa66d9dc)

### Navigating the Labyrinth of VLAN Trunking: A Proxmox and DD-WRT Troubleshooting Saga

![Article Figure](https://cdn-images-1.medium.com/max/800/0*-lq6HdNowNIXxRor.jpg)

Networking complex IT setups with limited hardware resources can be a real head-scratcher. My recent project involved a Proxmox VM server connected to a router via a single Ethernet cable. The challenge — I needed to host multiple virtual machines (VMs) on this server, each with distinct network access requirements.

#### Setting Up Network Zones for Our Virtual Machines

I set up specific network “zones” for our VMs:

- Internal Network Access: VMs in that network would have full access to my existing local network (to reach shared files or internal systems).
- External but Isolated: Some VMs needed to go online, but I made sure their network was kept separate from all our other networks. They could reach the internet, but not my private local systems.
- Private Network (VM-to-VM Only): The network is completely cut off from both our local network and the internet. VMs connected to that network would only be able to reach each other.

Because I only had that one physical cable from my Proxmox server to my home router, I decided to try out VLAN Trunking (also known as 802.1Q) between them. This technical trick adds a small piece of information (a VLAN ID) to each Ethernet frame, telling it which “virtual network” it belongs to. This way, many different networks (or broadcast domains) can share that single cable without getting mixed up.

#### Proxmox: Setting the Stage for Virtual Networks

My virtualization platform, Proxmox VE, offers support for VLANs and trunking. The initial configuration on the Proxmox host seemed straightforward:

1. Tagging default bridge: Setting VLAN tag on default vmbr0 for VLAN 1
2. Dedicated VLAN Bridge: Then, I created a second bridge vmbr2 , dedicated to VLAN 3 traffic. This bridge was also linked to the same physical Ethernet port.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*8Mrt0UtQw3Z0YwVHA-dwqw.png)

At this point, my Proxmox server was ready to send and receive tagged VLAN traffic:

- vmbr0 (VLAN 1): Configured for general VMs needing both local network and internet access. Traffic is tagged for VLAN 1, which is connected to both the Internet and the local network.
- vmbr2 (VLAN 3): Dedicated for VMs requiring inter-VM communication and internet access, but crucially, isolated from the local network.

#### Solving the VLAN Chicken-and-Egg Problem Between Proxmox and DD-WRT

When configuring VLANs between a Proxmox server and a DD-WRT router, you can run into a chicken-and-egg problem. The issue arises because Proxmox and DD-WRT both rely on each other being correctly configured before network traffic can flow properly.

Once I apply VLAN tagging in Proxmox, the system begins sending tagged packets. However, if the DD-WRT router isn’t yet set up to handle those tags, it will reject the traffic — cutting off Proxmox’s web interface and network access.

On the flip side, if I configure DD-WRT first to expect tagged traffic, but Proxmox is still sending untagged frames, the router won’t accept that traffic either. In both cases, communication between the two breaks down.

To solve this, apply the VLAN configuration on Proxmox first. This will temporarily interrupt access to the web UI. Then, configure DD-WRT to accept VLAN-tagged traffic on the port connected to the Proxmox server. Once done, connectivity is restored, and the system functions as expected.

This approach works because Proxmox’s behavior after applying changes is predictable, allowing time for configuration steps carefully. This sequence avoids the deadlock and gets both systems talking with the correct VLAN tags.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*HIQG5MNRBw8gUBaVSZHaOw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ypRobDGTQEu1nUvBfwhJYg.png)

#### The Router: DD-WRT’s Hidden Challenges

With Proxmox prepared, the spotlight shifted to the router. The Ethernet port connecting to my Proxmox server had to be configured as a trunk port on the router. If the router doesn’t recognize the 802.1Q tags, it will just drop or misroute the traffic from that port.

My router runs DD-WRT, a powerful custom router firmware known for its extensive features. While DD-WRT nominally supports VLAN trunking, its configuration proved to be unexpectedly difficult, at least for my router model. The web UI offered options to “enable tagging” (aka “trunking”) and presented a matrix of checkboxes for assigning VLANs to ports. I enabled tagging for the port to which my server is connected and allowed tagged traffic for VLAN 1 and VLAN 3. Then, for VLAN 3, I configured:

- Network address, mask, and interface address, which is also the default gateway for the network
- DHCP service with a dedicated address range
- Enabled Network Address Translation (NAT) and WAN port routing to allow VMs on VLAN 3 to reach the internet.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*SXKUuVC2KoV9mPpiOEBuUg.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*tR4CDglLjj39oGj7vnOG1Q.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*oJAKx0bYqaWmeeZ2glJteA.png)

Everything appeared correct in the web interface, but upon activation, I hit a wall.

#### The Head-Scratching Begins: Diagnosis and Symptoms

While the Proxmox web interface remained accessible via VLAN 1 (indicating tagged traffic from vmbr0 was routed as expected), VLAN 3 traffic wasn’t going through:

- VMs connected to vmbr2 couldn't communicate with the Internet.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*9TEy1Ul6_EY0E81E9pwpzw.png)

- After inspection, it turned out that DHCP requests from my VMs were left unanswered
- Even after the static IP configuration, none of the VMs were able to ping the router

![Article Figure](https://cdn-images-1.medium.com/max/800/1*_STp_3GlQSnAVS1IVjvtsw.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*cL7n7dEl-iXrc6I4QQWMAg.png)

- The router’s IP address for VLAN 3 couldn’t be resolved even with ARP (arping)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*fGTHpOSnk3FPdjM6sBunGA.png)

Logging into the router via SSH and running “ifconfig” on the vlan3 interface offered a glimmer of insight: traffic was indeed reaching the VLAN 3 interface, but the router simply wasn’t responding to it. Packet counters were incremented for received packets, but the router seemed to be ignoring them entirely.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*tfUUCdXlbZneWbu4h_pUVw.png)

To pinpoint the issue further, I ran tcpdump on the Proxmox server's physical interface. The captures confirmed that VLAN tags were correctly applied, and VLAN-tagged Ethernet frames were being sent as expected from the server's perspective.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*abm9g2u8JC2SkG33sxU99w.png)

#### The Deep Dive: Unearthing DD-WRT’s Quirks

![Article Figure](https://cdn-images-1.medium.com/max/800/1*g3Fi5zA5emFlgVBanUO0Uw.png)

My frustrating journey led me deep into DD-WRT forums and documentation. I have learned about nvram a non-volatile memory that stores router configurations, including port assignments and VLAN settings. DD-WRT has a command line tool to directly access and change those settings. It’s also called nvram . Despite verifying that VLAN 1 was functional and attempting to replicate its settings for VLAN 3 using nvram commands, the problem persisted. The configurations seemed correct on paper, but the router’s behavior suggested otherwise.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dfUiJIuLSKi74kEHOLm2jA.png)

Research led me to swconfig a powerful command-line utility. Unlike nvram , which stores settings for later use, swconfig provides direct access to and control over the router's internal switch chip (also called CPU). This chip is the brain that actually manages how physical ports handle network traffic and VLAN tags in real-time.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*U_57_OaGPsxV6kE13B1DIQ.png)

Inspecting swconfig 's output proved to be the breakthrough. While VLAN 3 was technically present in the router's overall configuration, swconfig revealed that its VLAN port assignments were incorrectly applied by the router’s operating system. The settings I thought were active (and correctly stored in nvram ) simply weren't configured on the switch hardware.

Manually adjusting the VLAN settings using swconfig —explicitly tagging the server's port as a trunk port for VLAN 3 at the switch chip level—finally resolved the issue. Almost instantly, VLAN 3 traffic flowed as expected: pings received replies, and ARP messages were successfully processed.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*cWEj-TJPx1ymE8d0--5stQ.png)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*k8h_Muphph1DUchAclYIlw.png)

This discovery led to a critical realization: the correct settings were being stored in nvram , but DD-WRT’s initialization sequence wasn’t consistently applying them to the underlying switch hardware on boot.

The definitive solution was to leverage DD-WRT’s startup script feature. This is a block of shell commands stored in nvram that are executed automatically during each router reboot. By adding the necessary swconfig commands to this script, I ensured that VLAN 3 settings were correctly applied every time the router powered on.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*ukeji9r5JooYNmwaa_SZDA.png)

#### Conclusion

My perseverance paid off. I now have a secure network setup where DD-WRT manages traffic from my server for both VLAN 1 and VLAN 3 through a single trunk port. VLAN 3 is isolated from other VLANs, yet with having internet access. Furthermore, DHCP is distributing separate IPs from IP address ranges for different VLANs, providing network segmentation and control.

This experience really showed me that it’s not just about knowing the theory behind networking — it’s about understanding how things work in the real world, with actual hardware and firmware. The little quirks and differences between systems can make a big difference, and that’s where hands-on learning really matters.
