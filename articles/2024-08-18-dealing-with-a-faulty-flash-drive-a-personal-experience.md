---
title: "Dealing with a Faulty Flash Drive: A Personal Experience"
date: 2024-08-18
medium_url: https://medium.com/@rabbit_knight/dealing-with-a-faulty-flash-drive-a-personal-experience-ed6207a2efb2
---

> **Original Published Article on Medium:** [Dealing with a Faulty Flash Drive: A Personal Experience](https://medium.com/@rabbit_knight/dealing-with-a-faulty-flash-drive-a-personal-experience-ed6207a2efb2)

![Article Figure](https://cdn-images-1.medium.com/max/800/1*LmUklnoAUwuaGz2PKomD1w.jpeg)

### Dealing with a Faulty Flash Drive: A Personal Experience

#### Introduction

Creating a bootable USB drive should be a straightforward task, but it can turn into a frustrating experience, especially when dealing with a faulty flash drive. In this article, I will share my personal experience and the steps I took to troubleshoot and resolve issues with a cheap flash drive that I initially used to create a bootable device. While I cannot guarantee these methods will work for everyone, I hope you find something useful for your own situation.

#### The Beginning: Creating a Bootable Drive

Let’s start from the beginning. My goal was to create a bootable drive to install Ubuntu on my computer. In preparation, I backed up all data from the flash drive and then unmounted all mounted partitions on that drive using console commands:

```
$
> 
cat
 /proc/mounts | grep /media | awk ‘{
print
 
$1
}’ | sudo xargs umount
```

Then, I tried to use my usual byte-copy command dd to “burn” the Ubuntu ISO I had downloaded onto my flash drive (it was /dev/sdb in my case):

```
$
> sudo 
dd
 
if
=Downloads/ubuntu.iso of=/dev/sdb oflag=direct status=progress
```

Unfortunately, the output was discouraging: “No medium found.”

#### The Issue: Recognizing the Media

In a nutshell, when I unplugged the drive and plugged it in again, it failed to recognize the media. Essentially, the kernel indicated that it couldn’t read data from certain parts of the disk. I suspected that the controller inside the flash drive was trying to mask its deficiencies by implementing non-standard hacks. So, when using standard tools to re-partition or re-write the flash drive, everything went sideways.

Spoiler alert: after some research, I confirmed my suspicion that the flash drive contained a cheap controller incapable of handling partition table (GPT in my case) rewrites correctly.

#### Attempted Solutions: Ejecting and Mounting

Next, I tried various methods suggested online, such as virtually ejecting the device. Interestingly, USB flash drives support the same API as old CD-ROMs, allowing them to be ejected without physically unplugging them. This essentially cuts power to the device while it remains visible to the system. So, I attempted to close this “virtual tray” using the following command:

```
$
> sudo eject /dev/sdb ; sudo eject --trayclose /dev/sdb
```

Then, I checked the detected block devices with the lsblk command — nope, no partitions were found…

I also tried manually mounting the flash drive using the udisks tool, a daemon that manages system block devices. I used the following command:

```
$
> udisksctl mount -b /dev/sdb
```

One notable feature of udisks is its ability to allow unprivileged users to control removable media, automatically handling mounting and unmounting. But this didn’t work either.

I also considered the possibility that udisks might be preventing my device from being recognized if it was unmounted. Therefore, I decided to stop the udisks SystemD service unit:

```
$>
 systemctl stop udisks2
```

Unfortunately, this had no effect either. When I re-plugged my flash drive, the system still wouldn’t recognize it and lsblk didn’t show any partitions related to that device.

#### The Solution: Zeroing Out the Disk

Finally, I decided to use the dd command to zero out the first 512 bytes of the disk medium, effectively erasing the Master Boot Record (MBR) and disk label by setting them to zeros:

```
$> sudo dd 
if
=
/dev/
zero 
of
=
/dev/
sda bs=
446
 count=
1
```

The command still threw an error like “write failed.” However, after doing this and re-plugging the flash drive, it was successfully detected by the system. I should caution you, though, that this solution is not entirely reliable. After multiple attempts to plug and unplug the drive, it was only detected about 30% of the time.

In conclusion, what ultimately worked was completely rewriting the MBR with zeros, which did something to the USB controller and allowed the Linux operating system to properly recognize the flash drive. After that, I was able to re-run my initial “burn” command without any issues:

```
$
> sudo 
dd
 
if
=Downloads/ubuntu.iso of=/dev/sdb oflag=direct status=progress
```

#### Final Thoughts

If you’re dealing with a cheap flash drive, be aware that altering or realigning the partition table might render the drive completely unresponsive. This doesn’t necessarily mean the flash drive is broken; rather, it’s a flaw in its design. While this issue shouldn’t occur with a well-made flash drive, it’s important to know that a faulty drive might not be entirely useless — it just requires a bit of extra care and troubleshooting.
