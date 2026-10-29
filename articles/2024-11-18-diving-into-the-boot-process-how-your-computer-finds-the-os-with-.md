---
title: "Diving into the Boot Process: How Your Computer Finds the OS with MBR"
date: 2024-11-18
medium_url: https://medium.com/@rabbit_knight/diving-into-the-boot-process-how-your-computer-finds-the-os-with-mbr-40f9ab452e30
---

> **Original Published Article on Medium:** [Diving into the Boot Process: How Your Computer Finds the OS with MBR](https://medium.com/@rabbit_knight/diving-into-the-boot-process-how-your-computer-finds-the-os-with-mbr-40f9ab452e30)

### Diving into the Boot Process: How Your Computer Finds the OS with MBR

![Article Figure](https://cdn-images-1.medium.com/max/800/1*CyIkdXq-QMtojlR34Um2mg.jpeg)

You know how sometimes you stumble upon a topic, and it just sticks with you? That happened to me over the weekend. I came across an article about the Master Boot Record (MBR) , and I just had to dig deeper. So, I sketched out a little diagram to make sense of it all, and now I’m here to share what I’ve learned. Let’s break it down in a way that’s easy to understand.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*cIB-GyoSlZWcKp7FDwEKuA.jpeg)

#### Starting from the Very Beginning

When you turn on your computer, the first thing that happens is that the CPU wakes up and starts executing instructions from a default address. For simplicity, let’s call it 0x00000000 (basically, it’s the very first address in hexadecimal). This address is found in ROM , where the BIOS (Basic Input/Output System) lives.

Think of the BIOS as the computer’s “morning routine.” It runs through its usual checks to make sure everything is working, like a quick health check for all the hardware. This process is known as POST (Power-On Self-Test) . Once it’s satisfied that everything is good to go, it moves on to the next task: finding something to boot up.

#### The Hunt for a Bootable Disk

If your computer is using the old-school BIOS/MBR boot method (instead of the newer UEFI ), the BIOS starts searching for the main disk it can boot from.

It looks for the first sector on the primary disk, known as the Master Boot Record (MBR) code . This MBR is only 512 bytes long — super tiny. But it holds some important code that helps the system to find the bootloader. Basically, it’s like a mini-map pointing to the next step.

#### What’s Inside the MBR code?

So, what does the MBR code actually do? Once the BIOS finds this MBR sector, it loads that 512-byte chunk of code into the RAM and hands over control to it. Since it’s such a small program, the MBR code doesn’t do much on its own. Its main job is to look at the partition table , which is also stored in the MBR.

The partition table lists all the partitions on your disk — like a directory for where everything is stored. It can support up to four primary partitions (or three primary and one extended partition). The MBR code scans through this table to find the active partition , which is the one marked for booting.

#### Next Up: The Bootloader

Once the MBR code finds the active partition, it reads data from the start of that partition, looking for the bootloader. It expects to find the bootloader address (the LBA number) on the disk. The bootloader is another, slightly bigger program that’s responsible for loading your operating system. The MBR code passes the flag to the bootloader, saying, “Okay, your turn to take over!”

The bootloader gets loaded into RAM and starts executing. Its job? To load the operating system’s kernel and all the essential components needed to get your system up and running.

#### Different Types of Bootloaders

Not all bootloaders are the same. For example:

- Windows Bootloader : This one is designed specifically for Windows. It’s pretty straightforward and isn’t designed to load other operating systems.
- Linux Bootloaders (like GRUB) : If you’re a Linux user, you’ve probably heard of GRUB (GRand Unified Bootloader) . It’s flexible and can be set up with a menu that lets you choose between different operating systems.

#### Legacy BIOS vs. Modern UEFI

Everything I’ve described so far is part of the old-school BIOS/MBR process. But if your system uses UEFI (the newer standard), things are a bit different. UEFI uses the GPT (GUID Partition Table) instead of MBR. It doesn’t rely on that tiny 512-byte sector and comes with some upgraded features — like support for bigger disks and more partitions.

And that’s the gist of how your computer finds its way to the operating system when it’s turned on! I hope this breakdown makes the boot process a little clearer.
