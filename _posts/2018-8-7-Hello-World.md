---
layout: post
title: Using hdparm To Get The Most Out Of Your USB Drives
---

Recently I replaced an internal 256GB SSD drive inside my laptop for a new one terabyte SSD which really made that machine much more useful. I ordered a case for the drive and plugged it in and saw that it had only 238 GB of available space. I just had to see if I could not take back some of that space and wipe all of the hidden spaces of any vendor settings or malware that may be on the drive. I have had some bad experiences doing this before but I was not ready to give up.

Just about any Linux distribution I have ever used has installed, or easily obtainable from the package manager, a hard drive tool called hdparm. This utility shows all the drive information and includes parameters that can be enabled or disabled including smart controls, power management, sleep settings, and caching.

For this example I wiped the drive initially with the dd command writing all zeros to the free drive space (this does not wipe the HPA or DCO hidden spaces however). Next I used gparted to reformat the drive to exfat which works on the Mac, Linux, and Windows and allows files sizes greater than 4GB to be written to it. Sure ext4 or maybe even NTFS would be better if only they were cross platform.

There is nothing special about using dd to wipe the drive. Many other tools are available but it is as good as any. First query the drive path.

$ sudo ifconfig

Assume for the example it is /dev/sdb then unmount the drive and use sudo when working with any block device.

$ sudo umount /dev/sdb

$ dd if=/dev/zero of=/dev/sdb

This will take quite a while and there will be no progress bar, just a cursor to stare at so be aware of that. If you need to stop it for any reason there is no harm done to the drive.

Now to hdparm on Debian Linux, or one of its derivatives, install the application and update/upgrade the system.

$ sudo apt-get install hdparm

$ sudo apt-get update && sudo apt-get upgrade

I recommend opening the man page for hdparm in the shell window to have as an easy reference and to copy and paste some very long commands coming up. Now we are ready to query the drive even though it is not mounted but still in the USB slot. The system can still see the device and with hdparm we can find out a lot about the drive and change settings as needed. The option is capital -I (eye).

$ sudo hdparm -I /dev/sdb && sudo hdparm -I /dev/sdb > myssd-1.txt

It helps as you go along to keep a record of how the drive was set up at first so I wrote the initial settings to a file called myssd-1.txt designating the pre-manipulation state if things go wrong. Then name consecutive files myssd-2.txt etc. so you can use diff to see the differences. hdparm can be a dangerous tool and the manual states that you can lose data and destroy the drive. It is even possible to destroy your operating system! iIt is recommended to do a reboot after major changes to a drive so the system can remount the drive. So you have been warned!! Make full backups and disconnect unneeded devices beforehand. As for me I am working on a wiped drive so there is no data to lose. Also this is an extra drive and I really want to make this work on a device larger than a thumb drive.

The output for hdparm is fairly lengthy so you may need to pipe the command to less. There are sections of output and the last section has to do with security. The asterisk symbol before a setting means that the setting is enabled. Make sure the drive is not locked and not password protected (the defaults). Setting a password is only temporary so make it easy to remeber and a part of the command. Setting the password is necessary for secure erasure and other commands on the hidden spaces.

Wiping the drive can be done with hdparm also and includes all hidden spaces. Even though I have already wiped the visible portion 0f the drive with dd I still opted to wipe it again with hdparm. The command below sets a user password to 123456. Then the second command uses that password to begin a long secure erase. Once this has been completed repeat the hdparm -I command and see the password has been removed. Make sure the drive is not frozen or locked again.

$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

$ sudo hdparm -I /dev/sdb

Now on to the hidden areas. I recommend removing the HPA (Host Protective Area) which holds some manufacturer diagnostic tools and is used by the BIOS so that the drive is used correctly.

$ sudo hdparm -N /dev/sdb

The output will incude a fraction of current sector count divided by Max Sector Count. The numerator will be less that the Max Sector and that is what we will remedy by making them equal. We will disable to HPA so that we have the max sectors in our usable drive area. Use the following to calculate how much you can recover by disabling the HPA.

$ echo $(((denominator- numerator) * 512))

To make the numerator equivalent to the denominaotr temporarily we run the following replacing <123456789> below with the denominator. We do this as a test before the permanent command which includes a p before the sector number. You may need to even include the "yes-i-know-what-i-am-doing" option. 

$ sudo hdparm -N <123456789> /dev/sdb

$ sudo hdparm -N p<123456789> /dev/sdb

$ sudo hdparm --yes-i-know-what-i-am-doing -N p<123456789> /dev/sdb

To see the final result repeat the following command. The output should now show a numerator equal to the denominator prior to the change.

$ sudo hdparm -N /dev/sdb

$ sudo hdparm -I /dev/sdb && sudo hdparm -I /dev/sdb > myssd-2.txt

Now comapre myssd-1.txt with myssd-2.txt to make sure the changes are as expected. The drive should be unlocked and unfrozen. The Max Sector size should be larger in file 2.

I just kept going to the next step at this point because there are no partitions on the drive and rebooting and formatting the drive again just to check it would be a lot of trouble.

The second hidden space we will consider is the DCO or Device Configuration Overlay used by manufacturers for extra features that come with the drive. To query the space on the SSD we will examine the output of hdparm -I again or grep the myssd-1.txt file for the Device Configuration Overlay status. An enabled DCO has an asterisk next to it. Also look for "Security not Locked not frozen". The drive needs to be unlocked and unfrozen to proceed.

$ sudo hdparm -I /dev/sdb

$ grep verlay myssd-1.txt

Now see what features can be altered on the drive and how much space can be added to the SSD if the DCO is wiped. The size will be in terms of 512 byte sectors as before however I was unable to discover the sizes of the HPA versus the DCO. I suspect both commands give the same basic Max Sectors in spite of the fact that we just disabled the HPA beforehand.

$ sudo hdparm --dco-identify /dev/sdb

See the number of "Real max sectors:" and multiply by 512 bytes per sector. This number should be larger that the value given in the myssd-1.txt file and and my SSD it added over 20GB to the size of the drive. I want that space and I want that space under my control and wiped free of any manufacturer apps and any malware that might be there.

Again we perform a none permanent DCO Restore to see how the drive reacts. Read all the warnings. Yes you can destroy the drive if you make this permanent so based on this ouput you decide if you want to risk it. Once you have made the decision perform the permanent command to restore the DCO drive area to your usable drive space with the second command that includes the claim you know what you are doing.

$ sudo hdparm --dco-restore /dev/sdb

$ sudo hdparm --yes-i-know-what-i-am-doing --dco-restore /dev/sdb

Repeat the command to see what the reult is and write to a file so we can compare the baseline values with the current condition of the drive.

$ sudo hdparm -I /dev/sdb && sudo hdparm -I /dev/sdb > myssd-3.txt

Finally time to reboot and see if the drive will be seen by the system so we can write a partition table to the drive and partition it again.

Nothing worked. I considered trashing the drive. I will try though to summarize what finally worked but I am unsure what really was the most important step. The drive was unusable for about 2 weeks and I tried various laptops with 3 different operating systems. Interestingly only the MacOS could see the drive but I could not partiton it. On linux the output of $ fdisk -l would just hang after my other usable drives. I then considered other tools and chose cfdisk and it showed the drive when fdisk would not. I wrote a partition table to it successfully then a FAT32 partition. Reboot. I tried this 2-3 times. At some point I ran hdparm again and checked parameters.

$ sudo hdparm -I /dev/sdb && sudo hdparm -I /dev/sdb > myssd-4.txt

Then I performed the password set and secure erase as above again.

$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

$ sudo hdparm -I /dev/sdb

And then I used cfdisk again to partition the drive then all was good. I formatted the drive to exfat where I wanted it in the first place and everything worked. Now the 256GB SSD that had only 238GB of space had the full 256GB. Was this worth all the hassle? To me it was. I learned a lot. In the future I will reboot more often. I will set the password then unset it before and after each major step. I will also use cfdisk. I am unsure why that worked better than fdisk or even if it did but it seemed more in tune with the drive. I did not have any OS problems and that would be unfortunate if it happened but if you are worried about that then maybe starting your laptop with say Kali on a USB would be a better alternative.
