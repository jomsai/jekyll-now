---
layout: post
title: Using hdparm To Get The Most Out Of Your USB Drives
---

Recently I replaced an internal 256GB SSD drive inside my laptop for a new one terabyte SSD which really made that machine much more useful. I ordered a case for the drive and plugged it in and saw that it had only 238 GB of available space. I just had to see if I could not take back some of that space and wipe all of the hidden spaces of any vendor settings or malware that may be on the drive. I have had some bad experiences doing this before but I was not ready to give up.

Just about any Linux distribution I have ever used has installed, or easily obtainable from the package manager, a hard drive tool called hdparm. This utility shows all the drive information and includes parameters that can be enabled or disabled including smart controls, power management, sleep settings, and caching.

For this example I wiped the drive initially with the dd command writing all zeros to the free drive space (this does not wipe the HPA or DCO hidden spaces however). Next I used gparted to reformat the drive to exfat which works on the Mac, Linux, and Windows and allows files sizes greater than 4GB to be written to it. Sure ext4 or maybe even NTFS would be better if only they were cross platform.

There is nothing special about using dd to wipe the drive. Many other tools are available but it is as good as any. First query the drive designation.

$ sudo fdisk -l

The fdisk command will show total bytes and sectors on the device so record these numbers. To
calculate the number of gigabytes on the drive.

$ echo $((256060514304/(1024 * 1024 * 1024)))

Approximately 238GB pof 256GB.

The drive should have the following;

$ echo $((256 * 1024 * 1024 * 1024))

274877906944 bytes instead of the visible bytes shown above as 256060514304.

Assume for the example it is /dev/sdb then unmount the drive and use sudo when working with any block device. This step is otional as hdparm has a secure erase feature but I plan to do both.

$ sudo umount /dev/sdb

$ sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

This will take quite a while. If you need to stop it for any reason there is no harm done to the drive.

Now to install hdparm on Debian Linux, or one of its derivatives, install the application and update/upgrade the system.

$ sudo apt-get install hdparm

$ sudo apt-get update && sudo apt-get upgrade

I recommend opening the man page for hdparm in the shell window to have as an easy reference and to copy and paste some very long commands coming up. Now we are ready to query the drive even though it is not mounted but still in the USB slot. The system can still see the device and with hdparm we can find out a lot about the drive and change settings as needed. The option is capital -I (eye). The -v option adds verbosity.

$ sudo hdparm -Iv /dev/sdb | tee myssd-1.txt

It helps as you go along to keep a record of how the drive was set up at first so I wrote the initial settings to a file called myssd-1.txt recording the pre-manipulation state if things go wrong. I named consecutive files numbered as myssd-1.txt etc. hdparm can be a dangerous tool and the manual states that you can lose data and destroy the drive. It is even possible to destroy your operating system! It is recommended to do a reboot after major changes to a drive so the system can remount the drive. So you have been warned!! Make full backups and disconnect unneeded devices beforehand. As for me I am working on a wiped drive so there is no data to lose. Also this is an extra drive and I really want to make this work on a device larger than a thumb drive.

The output for hdparm is fairly lengthy. There are sections of output and the last section has to do with security. The asterisk symbol before a setting means that the setting is enabled. Make sure the drive is not locked and not password protected (the defaults). Setting a password is only temporary so make it easy to remember. Setting the password is necessary for secure erasure and other commands on the hidden spaces.

Wiping the drive can be done with hdparm also and includes all hidden spaces. The command below sets a user password to 123456. Then the second command uses that password to begin a long secure erase. Once this has been completed repeat the hdparm -I command and see the password has been removed. Make sure the drive is not frozen or locked again.

$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

$ sudo hdparm -Iv /dev/sdb | tee myssd-2.txt

Now on to the hidden areas. I recommend removing the HPA (Host Protective Area) which holds some manufacturer diagnostic tools and is used by the BIOS so that the drive is used correctly.

$ sudo hdparm -N /dev/sdb

The output will include a fraction consisting of; 

MAX SECTORS available on the disk now / native real MAX SECTOR count possibly available on the disk.

The numerator will be less which means you have space to reclaim otherwise there is nothing more to do.

My 256 GB SSD should have about 274877906944 bytes possibly available but as the echo commands above show I only can use 256060514304. Each sector is 512 bytes.

Use the following to calculate how much space you can recover in sectors by disabling the HPA. Replace <denominator> and <numerator> with your devices values from the hdparm results you recieve.

$ echo $(((<denominator>-<numerator>) / 512))  ## Sectors available to reclaim

$ echo $(((274877906944 - 256060514304) / 512))  ## 36752720 sectors to reclaim in my example.

$ echo $(((274877906944 - 256060514304)))  ## 18817392640 bytes to reclaim.

$ echo $((18817392640 / (1024 * 1024 * 1024)))  ## 17GB to reclaim to go from
238 to 256GB.

Next do a TEST RUN with the <denominator> example using 274877906944 below. Again replace this value with your drives value.

$ sudo hdparm -N 274877906944 /dev/sdb | tee myssd-3.txt

Each time you run this option -N you need to hardware reset or power cycle the drive. See the manual page for hdparm -N option discussion. I see controversy about what this can mean. The most extreme way to do it would be eject the SSD and restart your computer. If anyone has any advice on this let me know but modprobe did not seem to be enough.

Once you have rebooted examine the myssd-3.txt file and make sure you have the values you expected. If so once you are ready to make the changes PERMANENT then run the command as below with a "p" in front of the desired sector count.

$ sudo hdparm -N p123456789 /dev/sdb

Current versions of the hdparm manual pages no longer discuss the
"--yes-i-know-what-i-am-doing" option shown below but would still use the option,
after a full drive recycle, if the previous command failed.

$ sudo hdparm --yes-i-know-what-i-am-doing -N p123456789 /dev/sdb

To see the final result repeat the following command. The output should now show that the sector values are equal.

$ sudo hdparm -N /dev/sdb

$ sudo hdparm -Iv /dev/sdb | tee myssd-4.txt

Now compare myssd-1.txt with myssd-4.txt to make sure the changes are as expected. The drive should be unlocked and unfrozen.

I just kept going to the next step at this point because there are no partitions on the drive and rebooting and formatting and mounting the drive again just to check it would be a lot of trouble but it is an option.

The second hidden space we will consider is the DCO or Device Configuration Overlay used by manufacturers for extra features that come with the drive. To query the space on the SSD we will examine the output of hdparm -I again or grep the myssd-1.txt file for the Device Configuration Overlay status. An enabled DCO has an asterisk next to it. Also look for "Security not Locked not frozen". The drive needs to be unlocked and unfrozen to proceed.

$ sudo hdparm -Iv /dev/sdb

$ grep verlay myssd-1.txt

Now see what features can be altered on the drive and how much space can be added to the SSD if the DCO is wiped. The size will be in terms of 512 byte sectors as before however I was unable to discover the sizes of the HPA versus the DCO. I suspect both commands give the same basic Max Sectors in spite of the fact that we just disabled the HPA beforehand.

$ sudo hdparm --dco-identify /dev/sdb

Again we perform a none permanent DCO Restore to see how the drive reacts. Read all the warnings. Yes you can destroy the drive if you make this permanent so based on this ouput you decide if you want to risk it. Once you have made the decision perform the permanent command to restore the DCO drive area to your usable drive space with the second command that includes the claim you know what you are doing.

$ sudo hdparm --dco-restore /dev/sdb

$ sudo hdparm --yes-i-know-what-i-am-doing --dco-restore /dev/sdb

Repeat the command to see what the result is and write to a file so we can compare the baseline values with the current condition of the drive.

$ sudo hdparm -Iv /dev/sdb | tee  myssd-5.txt

Finally time to reboot and see if the drive will be seen by the system so we can write a partition table to the drive and partition it again.

Nothing worked. I considered trashing the drive. I will try though to summarize what finally worked but I am unsure what really was the most important step. The drive was unusable for about 2 weeks and I tried various laptops with 3 different operating systems. Interestingly only MacOS could see the drive but I could not partiton it. On linux the output of $ fdisk -l would just hang after my other usable drives. I then considered other tools and chose cfdisk and it showed the drive when fdisk would not. I wrote a partition table to it successfully then a FAT32 format. Reboot. I tried this 2-3 times. At some point I ran hdparm again and checked parameters.

$ sudo hdparm -Iv /dev/sdb | tee > myssd-6.txt

I performed the password set and secure erase as above again.

$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

$ sudo hdparm -Iv /dev/sdb | tee > myssd-7.txt

And then I used cfdisk again to partition the drive then all was good. I formatted the drive to exfat where I wanted it in the first place and everything worked. Now the SSD was at the full 256GB. Was this worth all the hassle? To me it was. I learned a lot. In the future I will reboot more often. I will also use cfdisk. I am unsure why that worked better than fdisk or even if it did but it seemed more in tune with the drive. I did not have any OS problems and that would be unfortunate if it happened but if you are worried about that then maybe starting your laptop with linux on a bootable USB would be a better alternative.
