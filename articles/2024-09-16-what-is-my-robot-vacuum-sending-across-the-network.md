---
title: "What Is My Robot Vacuum Sending Across the Network?"
date: 2024-09-16
medium_url: https://medium.com/@rabbit_knight/what-is-my-robot-vacuum-sending-across-the-network-bc30f327c7a9
---

> **Original Published Article on Medium:** [What Is My Robot Vacuum Sending Across the Network?](https://medium.com/@rabbit_knight/what-is-my-robot-vacuum-sending-across-the-network-bc30f327c7a9)

### What Is My Robot Vacuum Sending Across the Network?

![Article Figure](https://cdn-images-1.medium.com/max/800/1*WRqpiMO_oHVvUnGLXR-x0g.png)

Have you ever wondered what your robot vacuum is transmitting across the network while you’re not watching? Follow along as I uncover what my Roborock is sending and to whom.

### Step-by-Step Process Using Wireshark:

#### 1. Find the MAC Address of Your Device

The first thing you need is your Roborock’s MAC address. For this, you can use netdiscover , a lightweight tool that identifies devices on your network by sending ARP requests to the subnet. To run it, use the following command in your terminal:

```
sudo netdiscover
```

This will scan your network and list all active devices. One of the entries should contain the word “Roborock.” Alongside it, you’ll see its IP address and MAC address.

Note: Make sure both your computer and the vacuum are connected to the same Wi-Fi network (and specifically, the same frequency band). Most networks operate on either 2.4 GHz or 5 GHz. The 2.4 GHz band offers longer range but slower speeds, while the 5 GHz band gives faster data rates over a shorter distance.

#### 2. Enable Monitor Mode on the Wi-Fi Adapter

To capture the raw Wi-Fi traffic, you need to put your Wi-Fi adapter into “monitor” mode. In this mode, the adapter won’t filter any radio-level frames, allowing Wireshark to capture the entire radio range.

2.1 Stop Network Manager

Before switching to monitor mode, ensure that no system software interferes with your Wi-Fi adapter. Most Linux distributions use NetworkManager to manage network connections. To find and stop it use the next commands.

1. Find the service:

```
sudo systemctl list-units --
type
=service | grep Net
```

This will list all running services that include “Net” in their name, so you can confirm what it’s called.

2. Stop the service:

```
sudo systemctl stop NetworkManager
```

![Disable NetworkManager](https://cdn-images-1.medium.com/max/800/1*PGqweXpafvpZyA288JA9Rg.jpeg)

*Disable NetworkManager*

This will cut off your internet connection. The Wi-Fi icon in your taskbar might disappear. From this point onward, you won’t have internet until you re-enable NetworkManager.

2.2 Switch to Monitor Mode

Now that NetworkManager is out of the way, it’s time to put the adapter into monitor mode.

1. Identify your Wi-Fi device:

```
sudo iw dev
```

This command will show all wireless interfaces along with their status. You’ll likely see that your device is currently in “managed” mode.

2. Disable the network interface:

```
sudo ifconfig <devname> down
```

3. Set the interface to monitor mode:

```
sudo iwconfig <devname> mode monitor
```

4. Reactivate the adapter:

```
sudo ifconfig <devname> up
```

5. Verify the adapter is in monitor mode:

```
sudo iw dev
```

#### 3. Capture Wi-Fi Traffic in Wireshark

Now it’s time to run Wireshark and capture the traffic.

3.1 Start Wireshark:

```
sudo wireshark -I
```

The -I flag ensures that Wireshark starts in an interactive mode, letting you choose the network interface.

In Wireshark’s main window, double-click the name of the Wi-Fi interface you just put into monitor mode. Wireshark will begin capturing raw traffic immediately.

You’ll likely see a flood of radio-level frames, which include everything from Wi-Fi beacons to acknowledgments (ACKs) and data frames. Most of these frames will be encrypted if your network uses WPA2 or WPA3 security.

3.2 Decrypt the Data

Since most Wi-Fi traffic is encrypted, Wireshark won’t show you much until you set it up to decrypt the packets. If you’re using WPA2 (with a password), here’s what to do:

1. Go to Edit > Preferences > Protocols > IEEE 802.11 .
2. Check the box for Enable decryption .
3. Add your network’s SSID and password under Decryption keys by selecting wpa-pwd and entering the key in the format <password>:<SSID> .

![Enable Decryption in Wireshark](https://cdn-images-1.medium.com/max/800/1*B3WHZJkQDVpOg7_k3jzclw.png)

*Enable Decryption in Wireshark*

3.3 Force EAPOL Renegotiation

Your router and vacuum cleaner use dynamic keys to encrypt data, which are negotiated when the vacuum connects to your Wi-Fi. To decrypt the traffic, you need to capture these EAPOL (Extensible Authentication Protocol Over LAN) frames during the handshake process.

Here’s how you can trigger this:

- Turn the vacuum cleaner off and then on again, forcing it to reconnect to the network.
- Alternatively, you can disconnect your vacuum from the Wi-Fi through the app and then reconnect it.
- As a last resort, restarting your Wi-Fi router will force all devices to renegotiate encryption keys, though this will disrupt your entire network.

Once the handshake occurs, Wireshark should be able to decrypt the subsequent traffic between the vacuum and the router.

#### 4. Filter Traffic by the Vacuum’s MAC Address

Use the vacuum’s MAC address to filter the captured packets, apply a filter in Wireshark: wlan.addr == <MACADDRESS>. This will show all the packets associated with your vacuum cleaner, filtering out unrelated traffic from other devices on your network.

#### 5. Analyze the Captured Data

After filtering, you should see various types of traffic, including decrypted TCP and UDP packets. Early on, you might still see some “802.11” management frames until Wireshark has fully captured and decrypted the EAPOL handshake. Once the handshake is captured, the real data starts to show.

In my case, I found that my vacuum was sending encrypted data to several IP addresses associated with Amazon Web Services (AWS) Hadoop. Hadoop is often used for distributed storage and processing of big data. Companies use it to analyze vast amounts of information by clustering multiple computers to work on the same dataset in parallel.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*dFDQqlttymseV1ZsnqZaRA.png)

What you choose to do with this information is up to you. Maybe it’s nothing to worry about, or maybe it’s worth a second look. Either way, it’s always interesting to see what’s happening on your network behind the scenes.
