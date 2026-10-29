---
title: "Hunting Open Ports: A Real-World nmap & Windows Investigation"
date: 2025-06-21
medium_url: https://medium.com/@rabbit_knight/hunting-open-ports-a-real-world-nmap-windows-investigation-e7291fe8759b
---

> **Original Published Article on Medium:** [Hunting Open Ports: A Real-World nmap & Windows Investigation](https://medium.com/@rabbit_knight/hunting-open-ports-a-real-world-nmap-windows-investigation-e7291fe8759b)

### Hunting Open Ports: A Real-World nmap & Windows Investigation

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Cc1zPXUKMgzdeBFJ8Wp9eQ.jpeg)

I recently dove into a small but eye-opening home lab experiment. The goal? To use basic tools like nmap to identify open ports and understand what services are running.

The Setup: Two Machines, One Network

- Linux box running CachyOS
- A standard Windows 11 machine

I wanted to check for vulnerabilities using nmap , and this is where things got interesting.

#### Step 1: Scanning the Linux machine

First, I opened a terminal on my Linux machine and ran:

```
nmap --script vuln [Linux Machine IP]
```

This command uses nmap’s pre-packaged vulnerability detection scripts ( --script vuln ) to scan the machine. --script vuln option tells nmap to run all nmap scripts from the “vulnerability” category against the target system, checking for common security weaknesses automatically. Instead of just listing open ports, it provides insight into potential risks linked to those services. It’s like having a quick security scanner built right into your port scan. Another handy nmap feature is the -O option that enables OS detection. Running nmap -O [target IP] allows you to identify the operating system version running on the target machine. It’s also called “fingerprinting”. This is very useful because knowing additional details about the target system’s OS helps you understand what vulnerabilities might apply and tailor your troubleshooting or security measures accordingly. Try it.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*eWKNk3CpO-tfd-NKP7rtfA.png)

Back to output. Everything came back clean. All ports were closed. No services exposed. So far, so good

#### Step 2: Scan the Windows Machine (Now We’re Talking)

Next, I turned Nmap toward the Windows machine:

Still on my Linux machine and ran:

```
nmap --script vuln [Windows IP]
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*K6DmDbbFh9e0yX7_fVZNiw.png)

While no known vulnerabilities were found (at least known to the NMAP scripts), this time, I saw a few ports open. One in particular caught my eye:

```
PORT     STATE SERVICE
80/tcp   open  http
```

Wait… Port 80? That’s typically used for web servers and it’s unencrypted. Why would a regular Windows 11 machine be running one?

#### Step 3: Confirm What’s Running — Browser & Curl Test

To see what was actually hosted on that port, I first opened a browser and typed in http://<TARGET_IP_ADDRESS>:80

![Article Figure](https://cdn-images-1.medium.com/max/800/1*-y_hegEKDpjgZib7mWgssA.png)

A basic default Web page loaded — so yes, Windows was running a web service. But just to double-check from Linux, I ran:

```
curl http://192.168.X.X
```

curl is just a CLI-based HTTP client that can fetch raw web content without any rendering. And sure enough, it returned the same HTML content. That confirmed it — something was hosting serving HTML from my Windows machine.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Rfk4pxsCN9GabiwdBx5KTQ.png)

#### Step 4: Find Out What’s Using Port 80

Time to dig deeper. I opened Command Prompt on Windows as Administrator and typed:

```
netstat -aon | findstr :80
```

This command shows:

- All listening ports ( -a )
- The owning process ID ( -o)
- Active connections ( -n for numeric display)

The output?

![Article Figure](https://cdn-images-1.medium.com/max/800/1*otYf9L5AlecyKxtDgQN9Sw.png)

The service listening on port 80 was tied to PID 4. And if you’ve used Windows before, you know that PID 4 usually points to “System” — a generic label for system-managed processes.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*EF-WJPQebqio3eWT69rNmw.jpeg)

#### Step 5: Identify the Actual Service

From there, I opened services.msc and scrolled through the running services. I found that World Wide Web Publishing Service (W3SVC) was running — this is part of IIS (Internet Information Services), Windows’ built-in web server.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*iBVqKlZ74YUslYLFQlRugA.jpeg)

I wasn’t hosting anything on this machine, so this was just taking up space and exposing a port for no good reason.

#### Step 6: Disabling IIS and Shutting That Door

Here’s how I cleaned it up:

Stop the Service

1. Run services.msc
2. Find World Wide Web Publishing Service
3. Right-click → Properties
4. Set Startup type to Disabled
5. Click Stop, then OK

![Article Figure](https://cdn-images-1.medium.com/max/800/1*l6i6hlsURsppScUGRGhubA.jpeg)

Uninstall IIS (Optional, but cleans things up for good)

1. Go to Control Panel → Programs and Features
2. Click Turn Windows features on or off
3. Uncheck Internet Information Services

![Article Figure](https://cdn-images-1.medium.com/max/800/1*aK53Dob3Eqqd-UUUfuENGA.jpeg)

Then I rebooted the system.

#### Step 7: Scan Again to Make Sure It’s Gone

Back in Linux, I ran:

```
nmap --script vuln [Windows IP]
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*EtJQiOFeT6RXk8UNJS9wgQ.png)

This time, port 80 was no longer listed as open — mission accomplished.

#### What I Took Away from This

This wasn’t some huge exploit, but it was a perfect example of how open ports can quietly exist without you realizing it, even on a clean install of Windows. IIS had been running the whole time, serving up who-knows-what on port 80.

And the best part? It didn’t require any expensive tools or advanced setups — just curiosity and two machines on the same network.

In addition to disabling IIS, I also took the time to review and disable several other non-essential Windows services that were listening on open ports. These ports weren’t needed for my setup and were just increasing my attack surface. I didn’t want to stretch this article too long, so I’ll cover that cleanup process in a future post.

Stay curious, and keep scanning!
