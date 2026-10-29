---
title: "The Storage Train"
date: 2024-09-03
medium_url: https://medium.com/@rabbit_knight/the-storage-train-b8b70b7eb9da
---

> **Original Published Article on Medium:** [The Storage Train](https://medium.com/@rabbit_knight/the-storage-train-b8b70b7eb9da)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*gOM16vq9qDuVxvESk9D5KA.jpeg)

### The Storage Train

Imagine our entire physical disk device as a grand train, with a powerful locomotive leading the way. When you first set up this storage train, it needs to be partitioned and formatted. This process ensures that the train cars (partitions) are ready for passengers (files). The locomotive represents the partitioning table. It welcomes anyone who boards the train. Just like any driver, it knows how many cars (partitions) the train contains and their overall length. Each car on our storage train corresponds to a partition on the disk. Just like separate train cars, each partition can be managed independently by its own file system — the conductor of our analogy. This conductor holds a manifest of all passengers and the layout of the car.

Let’s peek inside some of these passenger cars:

FAT16, FAT32, and exFAT: These file systems are like classic passenger cars — simple, widely used, and compatible across different platforms (Windows, macOS, and some Linux systems). They are simple to implement and manage but lack advanced features like security.

NTFS (Windows): NTFS is like a luxury compartment — feature-rich and robust. It supports large files, encryption, access control, and metadata.

HFS+ and APFS (macOS/Mac OS X): These file systems are the elegant dining cars of the storage world. HFS+ served well for years, but APFS (Apple File System) is the modern successor, optimized for SSDs and encryption.

ext4 (Linux): The ext4 file system is like a versatile sleeper car — reliable, widely used, and efficient. It’s the default for many Linux distributions.

Btrfs and ZFS: These are the futuristic Japanese cars — complex, powerful, and feature-packed. They support RAID, snapshots, and even span multiple partitions. Often used in virtualization environments.

The Swap Device

Picture the cistern as a massive water tank attached to the train. Unlike the other passenger cars, it doesn’t have assigned seats or organized compartments. When the operating system faces memory shortages, it unloads some data from RAM to the swap device. This gives the illusion of having more memory than physically available. Just like a cistern holds water temporarily, the swap device holds data temporarily. You can imagine this data as our extra luggage — it’s strange to put your luggage in the cistern, but for the sake of analogy, it works. All the extra luggage will be temporarily stashed inside the cistern. You can imagine that taking luggage from the cistern is very hard; that’s why accessing data from the swap device is slower than from RAM. Just as cisterns can vary in size, so do swap devices. You can adjust the cistern’s capacity (swap space) based on your needs. The swap device helps prevent memory overflow and keeps the train running smoothly.

Choosing Your Ride

Different file systems come with varying performance, security, and scalability characteristics. As a passenger on our storage train, it’s your choice which type of car you want to ride. Consider your specific needs:

- XFS (for MySQL Databases): If you’re running a MySQL database, hop into the XFS car — it’s your go-to. It’s like a well-engineered car, optimized for efficiency.
- FAT32 (for Simplicity and Firmware Integration): Seeking simplicity and seamless integration? Choose the FAT32 car. It’s like a reliable commuter train — familiar, straightforward, and compatible across platforms.
- Btrfs or ZFS (for Data Centers): These are futuristic Japanese cars — complex, powerful, and feature-packed. They support RAID, snapshots, and even span multiple partitions. Often used in virtualization environments.
