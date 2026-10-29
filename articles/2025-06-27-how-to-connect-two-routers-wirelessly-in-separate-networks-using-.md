---
title: "How to Connect Two Routers Wirelessly in Separate Networks Using DD-WRT Client Mode"
date: 2025-06-27
medium_url: https://medium.com/@rabbit_knight/how-to-connect-two-routers-wirelessly-in-separate-networks-using-dd-wrt-client-mode-a3b8821f79fa
---

> **Original Published Article on Medium:** [How to Connect Two Routers Wirelessly in Separate Networks Using DD-WRT Client Mode](https://medium.com/@rabbit_knight/how-to-connect-two-routers-wirelessly-in-separate-networks-using-dd-wrt-client-mode-a3b8821f79fa)

### How to Connect Two Routers Wirelessly in Separate Networks Using DD-WRT Client Mode

![Article Figure](https://cdn-images-1.medium.com/max/800/1*LzbfVTH0PRhPT3HG1uuldA.jpeg)

If you have two routers — one in use now (Router #1) and an old one (Router #2) running DD-WRT firmware, long forgotten in a dusty corner of your closet — you can connect Router #2 as a wireless client of Router #1. Why would you use such a configuration? Router #2 can become your connectivity gateway while you are travelling or performing complex lab setups while in the field. The most stable and secure way to do this is by configuring Client Mode on Router #2.

In this guide, I walk you through my actual setup and troubleshooting journey: from concept to configuration, through SSH hiccups, NAT, wireless settings, and tightening security.

#### Why Use DD-WRT Client Mode?

Before diving in, here’s why this setup is useful:

- Wireless Connection Between Routers: No Ethernet cable needed between Router #1 and Router #2. Client Mode allows Router #2 to connect wirelessly to Router #1 as a client. This extends connectivity to wired devices behind Router #2 without needing a physical cable between routers.
- Separate Subnets: Isolate devices for security. Devices on Router #2 won’t conflict with devices on Router #1’s network and will be shielded from it. That can be useful if you're performing your lab setups in the field: cafes, hotels, industrial sites, etc
- Internet Sharing: NAT (Network Address Translation) on Router #2 lets multiple devices share a single “WAN” IP (really an address of Router #2 as in the local network controlled by Router #1). Router #2’s devices can access the internet without exposing their IPs and ports to Router #1 and devices in its network.
- DD-WRT Flexibility: You can customize everything from firewall rules to startup scripts.

#### Step 1: Understand the Network Layout

Router #1(DD-WRT) :

- LAN IP: 192.168.1.1
- DHCP enabled: assigns IPs in 192.168.1.X
- Wi-Fi: WPA2-Personal + AES

Router #2 (DD-WRT):

- LAN IP: 192.168.200.1
- Wireless mode: Client
- DHCP: Enabled for 192.168.200.x
- NAT: Enabled
- Linux Machine is connected to Router #2 via Ethernet port

#### Step 2: Reset and Reconfigure DD-WRT (Router #2)

I started by resetting Router #2 to factory defaults to avoid hidden legacy settings. Then I logged back in at 192.168.1.1, changed the LAN IP to 192.168.200.X, and set the subnet mask to 255.255.255.0. This ensures Router #2 handles a different subnet than Router #1.

I enabled the DHCP server so devices connected to Router #2 can get IPs automatically in the 192.168.200.x range.

#### Step 3: Wireless Settings

Next, I navigated to: Wireless > Basic Settings

- Wireless Mode: Client
- SSID: Enter the exact name of Router #1 (case-sensitive)
- Channel: Matched to Router #1’s current channel

![Article Figure](https://cdn-images-1.medium.com/max/800/1*BZQ_oAUWzVBOQPEj9Dij8Q.png)

Then in: Wireless > Wireless Security. Match Router #1’s WPA2 Personal with AES encryption:

- WPA2-Personal
- AES encryption
- Entered Wi-Fi password

![Article Figure](https://cdn-images-1.medium.com/max/800/1*QIjYiyHzA6K1-ws2KRQTsw.png)

#### Step 4: Testing Connectivity

I connected a Linux machine via Ethernet to Router #2, and confirmed it received an IP from Router#2.

Test ping:

- 192.168.200.X (Router #2) → Success confirms LAN setup.
- 192.168.1.1 (Router #1) → Success confirms WAN access.
- 8.8.8.8 or any website → Confirms full internet access.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*gpgTkJ_51hKvCp6_ATt6pw.png)

#### Behind the Scenes

Here’s how this setup behaves:

- Router #2 connects wirelessly to Router #1 like any client device.
- Devices behind Router #2 get IPs in a separate subnet.
- NAT on Router #2 allows those devices to access the internet, while hiding their internal IPs.
- My Linux machine stays isolated from Router #1’s network but is fully connected online.

#### Accessing Router #2 via SSH

When I tried connecting with modern OpenSSH, I got the error. Some older DD-WRT versions still use outdated SSH algorithms. When connecting via modern OpenSSH, you may encounter: no matching key exchange method found

Solution: Temporarily Allow Weak Algorithms

```
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa root@192.168.200.X
```

Security note: These are insecure algorithms — don’t enable them system-wide. It’s okay to temporarily use them to connect to devices using old firmware (like the old DD-WRT version I mentioned). It’s recommended to perform a firmware update if available. Alternatively, it might be a more viable option to buy a new device!

![Article Figure](https://cdn-images-1.medium.com/max/800/1*k0FWWV4YQKCbe2XiCnPz1w.png)

#### CLI Checks and Security Hardening

Once logged in via SSH, I ran:

```
wl status         
# Should show connected SSID and mode (e.g., 802.11n)
arp -a            
# View ARP table; should list Router #1 and local clients
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*G6Y4Vo1o7qYj2ei9l6igKA.png)

This helped confirm that everything was working.

#### Checking Open Ports with netstat

To verify which ports are currently open on my DD-WRT router, I used the netstat command.

```
netstat -an
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*2aolIEcwrYCh22U_hJGGrw.png)

Among the expected open ports, I noticed port 23 (Telnet) was also active, which raised an alarm in my head. Since Telnet is not encrypted and insecure by default, the next step was to disable it. Telnet is a legacy protocol that is no longer considered safe to use.

#### How to Disable Telnet on DD-WRT via SSH

If you’re already connected to your router over SSH, follow these steps to disable Telnet:

```
nvram 
set
 telnetd_enable=0
nvram commit
reboot
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*n0jS5oJUvPMNVDoC70c2EQ.png)

After rebooting, the Telnet service should no longer be running.

If the telnetd_enable=0 setting doesn’t fully disable the Telnet service on your DD-WRT build, consider adding this to your router startup commands as a workaround: killall telnetd . This ensures the Telnet daemon is forcefully stopped after every boot.

#### Final Thoughts

In a previous project, I isolated networks within a single router using VLANs to create segmentation. That approach worked well for virtual isolation on a single device. [ Here’s the link .]

In this Saturday night project, I achieved true hardware-level separation between networks by using two physical routers configured with wireless Client Mode. Router #2 acts as a self-contained environment — with its own subnet, DHCP service, and connected devices — all routed through Router #1 for internet access, without merging the networks.

I believe this setup is ideal for:

- Isolating test devices or labs from your home/office LAN
- Running penetration testing tools or IDS without risking the main network stability
- Learning sandbox with full internet access but limited exposure
- Reusing older hardware (like that DD-WRT router sitting in your closet)

At the end of the day, for me, it’s about making routers talk safely, cleanly, and reliably — while learning some fun along the way.
