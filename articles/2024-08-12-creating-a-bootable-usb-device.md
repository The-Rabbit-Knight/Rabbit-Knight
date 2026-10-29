---
title: "Creating a Bootable USB Device"
date: 2024-08-12
medium_url: https://medium.com/@rabbit_knight/creating-a-bootable-usb-device-304d2a1dac4d
---

> **Original Published Article on Medium:** [Creating a Bootable USB Device](https://medium.com/@rabbit_knight/creating-a-bootable-usb-device-304d2a1dac4d)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*Hdn709E76y3us4F9WeX_wg.jpeg)

### Creating a Bootable USB Device

Creating a bootable USB device involves several key steps. Here’s my guide for the process:

#### Step 1: Download the ISO File

First, download the ISO file from your preferred Linux distribution’s official website. For this example, I’ll use the ISO file from the official Ubuntu website. Ensure you’re downloading the correct version for your system architecture (e.g., 64-bit or 32-bit).

#### Step 2: Prepare the USB Device

You’ll need a physical USB device to burn the downloaded ISO onto. Typically, the USB should be at least 8 gigabytes, though some larger distributions might require more space. I’m using a 128-gigabyte flash drive, which provides ample space for the ISO file. Note: If your USB device is faulty or of poor quality, it may fail during the process. I encountered this issue and will share a link on how I resolved it for reference later.

#### Step 3: Backup Data

Before proceeding, back up any important data that was stored on the USB device, as the process will erase everything on it. I backed up my data to my laptop to ensure nothing was lost.

#### Step 4: Unmount the USB Device

Before you can burn the ISO file to your USB device, you need to ensure it is unmounted. This prevents the operating system from interfering with the process and ensures a clean write.

To clarify, mounting refers to logically attaching a file system to a specific access point, making it accessible by the user. A file system is a hierarchy of directories located on an isolated unit of a physical storage drive called a partition.

To check which file systems are currently mounted, I used the df -h command in Linux. This command displays the disk space usage for all mounted file systems, including their mount points and corresponding devices. You can explore additional options with the help command ( -h stands for "human-readable format").

If you’re using Ubuntu or a similar GNOME-based system, you might see something like /media/<your_username>/<mount_point> . <mount_point> can be some intelligible name, or a UUID ( Universally Unique Identifier as a hexadecimal sequence). Depending on the desktop environment, it might appear differently. If you have only one flash drive connected, it will be easy to identify its mount point, as there will be just one entry in the /media/<your_username> folder.

Use the umount command to unmount the USB device. The umount command is used to manually unmount file systems on Linux and other Unix-like operating systems. You can specify either a disk name or device name (/dev/device_name) or the mount point’s absolute path. The command will accept both options.

So, in my case, the command looks like sudo umount /dev/dm-1

#### Step 5: Deactivate Device Mapper (if applicable)

If your USB device is encrypted or a part of a DeviceMapper volume, you need to deactivate it. DeviceMapper is a kernel subsystem framework that allows you to create virtual block devices over physical ones, often used for software RAIDs and LUKS (Linux Unified Key Setup) encryption. LUKS provides block device encryption, securing your data.

In practice, if you see dm-1 , dm-2 , etc., in the df -h output, it indicates that a device mapper was used. To deactivate a DeviceMapper volume, I used the following command:

sudo dmsetup remove -f /dev/dm-<number>

This command “closed” my encrypted volume and released resource locks on the underlying media.

#### Step 6: Burn the ISO to the USB Device

With the USB device unmounted and any device mappings deactivated, you can now burn the ISO file.

You might be wondering how to identify the device file for your flash drive in the /dev folder and how to ensure you're using the correct device and not, for instance, your root volume. If you know the model or maker of your flash drive, you can easily identify the device file by running:

lsblk -o model,name

The -o option specifies which columns of information to display. In this case, model shows the model name of each device, and name shows the device name (e.g., sda , sdb ).

Next, I used the dd command to perform a byte-for-byte copy from the ISO file to the USB device:

sudo dd if=/<path_to_ubuntu.iso> of=/dev/<device_name> bs=4M status=progress

- if=/<path_to_ubuntu.iso> : The if stands for "input file." Here, you specify the path to the ISO file that you want to write to the USB device. Replace /<path_to_ubuntu.iso> with the actual path to your ISO file.
- of=/dev/<device_name> : The of stands for "output file." This is the path to the USB device you want to write the ISO file to. Replace /dev/<device_name> with the correct device name for your USB drive.
- bs=4M : The bs stands for "block size." Setting this to 4M means that dd will read and write 4 megabytes of data at a time.
- status=progress : This option provides real-time feedback on the progress of the operation, showing you how much data has been written and the current speed.

#### Step 7: Verify the Bootable USB

Once the dd command completes, safely eject the USB device. You now have a bootable USB drive ready to use.
