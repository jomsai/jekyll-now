---
layout: post
title: Using hdparm To Get The Most Out Of Your USB Drives
---

Recently I replaced an internal 256GB SSD drive inside my laptop for a new one terabyte SSD which really made that machine much more useful. I ordered a case for the old drive and plugged it in and saw that it had only 238 GB of available space. I just had to see if I could not take back some of that space and wipe all of the hidden spaces of any vendor settings or malware that may be on the drive. I have had some bad experiences doing this before but I was not ready to give up.

Just about any Linux distribution I have ever used has installed, or easily obtainable from the package manager, a hard drive tool called hdparm. This utility shows all the drive information and includes parameters that can be enabled or disabled including smart controls, power management, sleep settings, and caching.

For this example I wiped the drive initially with the dd command writing all zeros to the available drive space (this does not wipe the HPA or DCO hidden spaces however). Next I used gparted to reformat the drive to exfat which works on the Mac, Linux, and Windows and allows files sizes greater than 4GB to be written to it. Sure ext4 or maybe even NTFS would be better if only they were cross platform.

There is nothing special about using dd to wipe the drive. Many other tools are available but it is as good as any. First query the drive designation.

	$ sudo fdisk -l

		Drive designation=sdb.
		Drive capacity 238.49 GiB or 256060514304 bytes.
		Drive capacity 500118192 sectors.

Some quick calculations to confirm that these values align with what I see on the desktop which that the drive capacity is 238GB but above says I have over 238Gib.

	$ echo $((256060514304/(1024 * 1024 * 1024)))
		Shows Total Available Bytes divided by 1 GiB=238GiB

	$ echo $((256060514304/512))
		Shows Total Available Bytes divided by 512 bytes/sector=500118192 sectors

The values are in Gibibytes. The box that came with the drive stated it was a 256GB one. If I mount the drive on the desktop and right click on it and select"properties" or "Get Info" it shows it is 238GB.

If you divide total bytes by 1GB you do obtain 256GB already in use. 

	$ echo $((256060514304/(1000 * 1000 * 1000)))
		Shows Total Avail. Bytes divided by 1GB=256GB

This is what is on the box. Maybe I do not have hidden spaces.

Alternatively how many bytes should there be on the drive for 256GiB?

	$ echo $((256 * (1024 * 1024 * 1024)))
		Shows Possible 274877906944 bytes using 1GiB

	$ echo $((256 * (1000 * 1000 * 1000)))
		Shows Possible 256060514304 bytes using 1GB

There are 274877906944 bytes possible but I only have 256060514304 bytes so really the box should say 256GiB drive. Therefore I do have some space to reclaim.

Now unmount the drive.

	$ sudo umount /dev/sdb

I plan on doing a complete wipe of the drive, or at least the visible available space with the dd command. Later I will also do a secure erase with hdparm which I understand wipes all the hidden areas.

	$ sudo dd if=/dev/zero of=/dev/sdb bs=1M status=progress

This will take quite a while. If you need to stop it for any reason there is no harm done to the drive.

Now to install hdparm on Debian Linux, or one of its derivatives, install the application and update/upgrade the system.

	$ sudo apt-get install hdparm

	$ sudo apt-get update && sudo apt-get upgrade

I recommend opening the man page for hdparm in the shell window to have as an easy reference and to copy and paste some very long commands coming up.

NOTE: The manual pages for hdparm is in flux. I no longer see the option --i-know-what-i-am-doing in the manual as a disclaimer before you performed a final permanent command. More on that later.

Now we are ready to query the drive even though it is not mounted but still in the USB slot. The system can still see the device and with hdparm we can find out a lot about the drive and change settings as needed. The option is capital -I (eye). The -v option adds verbosity.

	$ sudo hdparm -Iv /dev/sdb | tee myssd-1.txt

It helps as you go along to keep a record of how the drive was set up at first so I wrote the initial settings to a file called myssd-1.txt recording the pre-manipulation state if things go wrong. I named consecutive files numbered as myssd-1.txt myssd-2.txt etc.

hdparm can be a dangerous tool and the manual states that you can lose data and destroy the drive. It is even possible to destroy your operating system! It is recommended to do a reboot after major changes to a drive so the system can remount the drive. So you have been warned!! Make full backups and disconnect unneeded devices beforehand. Maybe use an old computer you no longer need. As for me I am working on a wiped drive so there is no data to lose. Also this is an extra drive and I really want to make this work on a device larger than a thumb drive.

The output for the hdparm -I command in the myssd-1.txt file is fairly lengthy. There are sections of output and the last section has to do with security. The asterisk symbol before a setting means that the setting is enabled. Make sure the drive is not locked and not password protected (the defaults). Setting a password is only temporary through 1 command and has to be set several times so make it easy to remember. Setting the password is necessary for secure erasure and other commands on the hidden spaces.

Wiping the drive can be done with hdparm as stated above and includes all hidden spaces. I assume this means once you have reclaimed these hidden spaces. I make this assumption because if it deleted hidden spaces (securely erasing them) then why would you need other commands?

The command below sets a user password to 123456. Then the second command uses that password to begin a long secure erase. Once this has been completed repeat the hdparm -I command and see the password has been removed. Make sure the drive is not frozen or locked again.

	$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

	$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

	$ sudo hdparm -Iv /dev/sdb | tee myssd-2.txt
 		...
		Security: 
				Master password revision code = 65534
				supported
			not	enabled
			not	locked
			not	frozen

The password has been removed as above by Security... "not enabled" now and the drive is not frozen or locked.

Now on to the hidden areas. I want to remove the HPA (Host Protective Area) which holds some manufacturer diagnostic tools and is used by the BIOS so that the drive is used correctly. For this see the -N option.

	$ sudo hdparm -N /dev/sdb

The output will include a fraction consisting of; 

MAX SECTORS available on the disk now / native real MAX SECTOR count possibly available on the disk.

The numerator should be less which means you have space to reclaim otherwise there is nothing more to do.

Use the following to calculate if you can claim any space at all, and how much you can recover in sectors by disabling the HPA. I recommend that you replace <denominator> and <numerator> with your devices values from the hdparm command on your terminal not the fdisk command.

	$ echo $(((<denominator>-<numerator>) / 512))
		Shows Number of 512 byte sectors available to reclaim.

	$ echo $(((274877906944 - 256060514304) / 512))
		Shows 36752720 sectors to reclaim in my example.

	$ echo $(((274877906944 - 256060514304)))
		Shows 18817392640 bytes to reclaim.

	$ echo $((18817392640 / (1024 * 1024 * 1024)))
		Shows about 17GiB to reclaim.

Next do a TEST RUN with the <denominator> example using 274877906944 below. Again replace this value with your drives value.

	$ sudo hdparm -N 274877906944 /dev/sdb | tee myssd-3.txt

Each time you run this option -N you need to hardware reset or power cycle the drive. See the manual page for hdparm -N option discussion. I see controversy about what this can mean on the internet. The most extreme way to do it would be eject the SSD and restart your computer. However you may not be able to mount or query the drive with fdisk -l. If anyone has any advice on this let me know but modprobe did not seem to be enough. I will call this a recycle of the drive in this discussion.

Once you have recycled the drive examine the myssd-3.txt file and make sure you have the values you expected. If so once you are ready to make the changes PERMANENT then run the command as below with a "p" in front of the desired sector count.

	$ sudo hdparm -N p274877906944 /dev/sdb

Current versions of the hdparm manual pages no longer discuss the
"--yes-i-know-what-i-am-doing" option shown below but I would still use the option,
after a full drive recycle, if the previous command failed.

OPTIONAL.

	$ sudo hdparm --yes-i-know-what-i-am-doing -N p274877906944 /dev/sdb

		...
		/dev/sdb
			setting max visible sectors to 274877906944
		...
			max sectors = 256060514304/274877906944
		...

Don't be confused here. This says the drive has been changed to the larger byte count and calls it sectors. Then it says HPA is enabled. But we just disabled it. We need to recycle the drive.

To see the final result repeat the following command or commands. The output should now show that the sector values are equal.

	$ sudo hdparm -N /dev/sdb

You may have to recycle the drive. If you do not want to risk it use hdparm -I.

	$ sudo hdparm -Iv /dev/sdb | tee myssd-4.txt

Now compare myssd-1.txt with myssd-4.txt to make sure the changes are as expected.

	BEFORE:  *	Host Protected Area feature set
	AFTER:      	Host Protected Area feature set

The second hidden space we will consider is the DCO or Device Configuration Overlay used by manufacturers for extra features that come with the drive. To query the space on the SSD we will examine the output of hdparm -I again or grep the myssd-1.txt file for the Device Configuration Overlay status. An enabled DCO has an asterisk next to it. Also look for "Security not Locked not frozen". The drive needs to be unlocked and unfrozen to proceed.

	$ sudo hdparm -Iv /dev/sdb

The following should show the line showing Device Configuration Overlay information.

	$ grep verlay myssd-1.txt
		*	Device Configuration Overlay feature set

Now see what features can be altered on the drive and how much space can be added to the SSD if the DCO is wiped with the --dco-identify option. The size will be in terms of 512 byte sectors as before however I was unable to discover the sizes of the HPA versus the DCO. I suspect both commands give the same basic Max Sectors in spite of the fact that we just disabled the HPA beforehand.

	$ sudo hdparm --dco-identify /dev/sdb

Again we perform a none permanent command with the --dco-restore option to see how the drive reacts. Read all the warnings. Yes you can destroy the drive if you make this permanent so based on this ouput you decide if you want to risk it. Once you have made the decision perform the permanent command to restore the DCO drive area to your usable drive space with the --dco-restore option.

	$ sudo hdparm --dco-restore /dev/sdb

The second OPTIONAL command includes the claim you know what you are doing.

	$ sudo hdparm --yes-i-know-what-i-am-doing --dco-restore /dev/sdb

Repeat the hdparm -I command to see what the result is and write to a file so we can compare the baseline values with the current condition of the drive. The goal is to remove the asterisk in the line;

	$ sudo hdparm -Iv /dev/sdb | tee  myssd-5.txt
		RESULT: 	Device Configuration Overlay feature set.
		(No * so NOT SET)

Success!

Finally time to reboot and see if the drive will be seen by the system so we can write a partition table to the drive and partition it again. Both hidden areas should have been reclaimed.

Nothing worked. The drive would not mount of course and the OS did not ask if I can format the USB device just plugged in. I considered trashing the drive. I will try though to summarize what finally worked but I am unsure what really was the most important step. The drive was unusable for about 2 weeks and I tried various laptops with 3 different operating systems. Interestingly only MacOS Disk Utility could see the drive but I could not partiton it. On linux the output of $ fdisk -l would just hang after my other usable drives. I then considered other tools and chose cfdisk and it showed the drive when fdisk would not. I wrote a partition table to it successfully then a FAT32 format. Reboot. I tried this 2-3 times. At some point I ran hdparm again and checked parameters.

	$ sudo hdparm -Iv /dev/sdb | tee > myssd-6.txt

I performed the password set and secure erase as above again.

	$ sudo hdparm --user-master u --security-set-pass 123456 /dev/sdb

	$ sudo hdparm --user-master u --security-erase 123456 /dev/sdb

	$ sudo hdparm -Iv /dev/sdb | tee > myssd-7.txt

And then I used cfdisk again to partition the drive again and all was good. I formatted the drive to exfat where I wanted it in the first place and everything worked. Now the SSD is mounted and shows the full 256GB capacity on the desktop.

Was this worth all the hassle? To me it was. I learned a lot. In the future I will recycle more often. I will also use cfdisk. I am unsure why that worked better than fdisk or even if it did but it seemed more in tune with the drive. I did not have any OS problems and the drive works great.

DISCLAIMER: I am not responsible for any HD, SSD, OS failures you may have after using the commands above. Every manufacturer device is different and some set parameters to perform better on one OS or another. If the initial query does not show what you expect abandon making any changes.
