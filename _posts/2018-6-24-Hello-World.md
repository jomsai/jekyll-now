---
layout: post
title: Using hdparm To Get The Most Out Of Your USB Drives
---

Recently I replaced an internal 256GB SSD drive inside my laptop for a new one terabyte SSD which really made that machine so much more awesome. I ordered a case for the drive and plugged it in and saw that it had only 238 GB of available space. I just had to see if I could not take back some of that space and wipe all of the hidden spaces of any vendor settings or malware that may be on the drive.

Just about any Linux distribution I have ever used has installed, or easily obtainable from the package manager, a hard drive tool called hdparm. This utility shows all the drive information and includes parameters that can be enabled or disabled including smart controls, power management, sleep settings, and caching.

For this example I wiped the drive initially with the dd command writing all zeros to the free drive space. Then I used gparted to reformat the drive to exfat which works on the Mac, Linux, and Windows and allows files greater than 4GB to be written to it. Sure ext4 or maybe even NTFS would be better if only they were cross platform.

There is nothing special about using dd to wipe the drive. Many other tools are available but it is as good as any. First query the drive path.

$ sudo ifconfig

Assume for the example it is /dev/sdb then unmount the drive and use sudo when working with any block device.

$ sudo umount /dev/sdb

$ dd if=/dev/zero of=/dev/sdb

This will take quite a while and there will be no progress bar so just know that. If you need to stop it for any reason there is no harm done to the drive.

Now to hdparm on Debian Linux, or one of its derivatives, install the application and update/ upgrade the system.

$ sudo apt-get install hdparm

$ sudo apt-get update && sudo apt-get upgrade

I recommend opening the man page for hdparm in the subshell to have as an easy reference and to copy and paste some very long commands coming up. Now we are ready to query the drive even though it is not mounted but still in the USB slot. The system can still see the device and with hdparm we can find out a lot about the drive and change settings as needed.

$ sudo hdparm -I /dev/sdb && sudo hdparm -I /dev/sdb > myssd-1.txt

It helps as you go along to keep a record of how the drive was set up at first so I wrote the initial settings to a file called myssd-1.txt designating the pre-manipulation state if things go wrong. Then name consecutive files myssd-2.txt etc. so you can use diff to see the differences. hdparm can be a dangerous tool and the manual states that you can lose data and destroy the drive. So you have been warned. As for me I am working on a wiped drive so no data to lose. Of course back up any files before using hdparm. Also this is an extra drive and I really want to make this work on a device larger than a thumb drive.

The output for hdparm is fairly lengthy so you may need to pipe the command to less. There are sections of output and the last section has to do with security. The asterisk symbol before a setting means that the setting is enabled. Make sure the drive is not locked and 

