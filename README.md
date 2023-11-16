
# STANDALONE
The following will not work unless you have first removed your computer's internal drive and replace it with your own internal or external ssd, if you continue without having done so, you will overwrite your main OSs efi/bios partition and potentially your nvram data, likely leaving you with an unbootable machine and a trip to the technician....
 
By doing so, one can completely bypass all of the issues and obstacles present such as:
* the potential overwrite of important data, directories and partitions potentially leading to an irrecoverable loss during installation.
* the overwrite of our host machine's efi partition resulting in an unbootable machine.
 
Since I completed this on a MacBook Air (13-inch, Early 2015 model), I will assume you'll do the same, although I believe this would work on any EFI bootable machine as long as you remove the internal drive. I used an 500 GB pcie 3.2 as the installation medium, and a portable 256G NVME 4.0x4 Blade SSD for the target drive in place of the Apple M.2 SSD.
 
I compiled this by sourcing through numerous different guides, blogs and reddit posts before I found a way that worked which once replicated has only really seemed to work this way which is probably due either memory constraints and a data bottleneck if using old hardware. Believe me it gets much more fast when wou plug an ssd directly into the board and install off of a usb 3.2+ via SATA. That is truly when the power of the Debian Installer is realized with Kali Linux. What I mainly used:
 
	https://www.kali.org/docs/installation/hard-disk-install/
	https://www.kali.org/docs/installation/btrfs/
	https://www.kali.org/docs/usb/usb-standalone-encrypted/
	https://www.gnu.org/software/grub/manual/grub/grub.html#Introduction-1
	https://superuser.com/questions/1705973/add-efi-partition-to-bios-menu-after-nvram-reset
	https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html
	https://bbs.archlinux.org/viewtopic.php?id=268460
	

 
Lastly, while I uncovered a few ways to get this done I am going to assume that you are doing exactly as I did and so from here on out I will refer to our target drive as nvme0n1 and its partitions as nvme0n1p1, nvme0n1p2, nvme0n1p3/nvme0n1p3_crypt/luks-aaaa-aa-aa-aa-aaaaaa, nvme0n1p4_crypt and nvme0n1p5_crypt and so on, so make sure to check your device ids etc before making any changes. Some machines will not recognize the drives as nvme0n1 and will register them as sdX unless mounted in an adapter and the other way around in some cases. Copy this file down and adjust it before you start, checking your drives with lsblk via command line first.
 
In Mac OS, power it down and carefully remove the internal drive. This will take care of /target/boot/efi being mounted to the wrong location.
 
Replace the internal drive with the new SSD we are using as our /target. Plug back in the power cable, our Kali installer USB and reboot our computer from the image.
 
Continue by selecting "Advanced Options" and then selecting "graphical expert installation guide.."
 
Select:

	advanced options
and then:

	graphical expert install
and wait for the installer to load a list options.
 
Begin selection of default install options and settings according to language, keyboard preferences etc.
 
When selecting additional components choose:
 
	cryptodisks udeb
	parted udeb
	low-mem install
	mbr udeb
	fdisk udeb
	rescue mode
 
And continue.

Ignore warnings when setting up the network if you are using an unsupported driver which is the case for my mac. So continue with:

	network name
	system name
	user name & passwords
	Install base
	Configure package manager
And select:

	cryptdisks
	fdiskudeb
	partedudeb
	low-mem
	mbrudeb
	rescuemode
Select:

	detect disks, then partition:
Select:

	manual partitioning.
 
While gdisk is great, the installer's built in partition editor does a great job on it's own.
* Select the free size and make a new one of a higher size, like 570 MB and use that partition for EFI.
* Again, select free size, this time allocating at least 5G (6.9 GB in my case) and selecting ext4 as the filetype with /boot as the mount point and noatime as the option.
* For your next partition, allocate at least 16.9 GB for swap space however select 'physical volume for encryption'.
* Select the remaining free space and allocate the remainder for a root partition and select 'physical volume for encryption'.
* At the top of the menu, select 'configure encrypted volumes'.
* Select 'create encrypted volume'
* Select the volumes you wish to encrypt, typically labeled with 'crypt'
* Select 'Finish', answer yes when it prompts you to write changes to the disk and fill out the password fields.
* There should now be two drives visible at the top of the partition editor display, one which is for swap and the other for root, select that and:
* Select 'use as', then choose 'seap' from the selections.
* Select the second drive and choose 'btrfs', select 'options' and choose 'noatime' format and finish.
* Lastly, choose 'finish partitioning and write changes to disk', then continue.
 
Now choose:

	Select and install software
deselect all but:

	XFCE
	Default Software.
and continue.

	Install Grub boot loader and select yes when asked if you want to force efi to the removable path. 
DO NOT allow it to change your NVRAM settings; I REPEAT: DO NOT ALLOW IT TO CHANGE YOUR NVRAM SETTINGS!

Don't install os-prober.
 
Finish with defaults.
 
Finish installing, remove the installation media and reboot into newly installed drive, continuing with the instructions below:
 
# Setting up BTRFS filesystems.

	sudo apt dist-upgrade
	sudo apt auto-remove -y
	sudo apt clean
necessary:

	sudo passwd	
Install some essential tools

	sudo apt update && sudo apt install btrfs-progs snapper snapper-gui grub-btrfs
Create the snapper configuration for the root filesystem "/"

	sudo cp /usr/share/snapper/config-templates/default /etc/snapper/configs/root
	sudo sed -i 's/^SNAPPER_CONFIGS=\"\"/SNAPPER_CONFIGS=\"root\"/' /etc/default/snapper
Prevent "updatedb" from indexing the snapshots, which would slow down the system

	sudo sed -i '/# PRUNENAMES=/ a PRUNENAMES = ".snapshots"' /etc/updatedb.conf

# Reconfigure lightdm to allow booting into read-only snapshots
	sudo sed -i 's/^#user-authority-in-system-dir=false/user-authority-in-system-dir=true/' /etc/lightdm/lightdm.conf
	sudo reboot now


Archive the directory elsewhere (on another device), and unmount it afterwards.
 
	sudo mount -oremount,ro /boot
	sudo mount -oremount,ro /boot/efi
	sudo chown <user> /tmp
	sudo install -m0600 /dev/null /tmp/boot.tar
	sudo tar -C /boot --acls --xattrs --one-file-system -cf /tmp/boot.tar .
	sudo umount /boot/efi
	sudo umount /boot
Wipe out the underlying block device (assumed to be /dev/nvme0n1p3 in the rest of this sub-section).
 
	sudo dd if=/dev/urandom of=/dev/nvme0n1p2 bs=1M status=none
	#	dd: error writing '/dev/nvme0n1p2’: No space left on device
Format the underlying block device to LUKS1. (Note the --type luks1 in the command below, as Buster’s cryptsetup(8) defaults to LUKS version 2 for luksFormat.)
 
	sudo cryptsetup luksFormat --type=luks1 /dev/nvme0n1p2
 
	#	WARNING!
	#	========
	#	This will overwrite data on /dev/nvme0n1p2 irrevocably.
	
	#	Are you sure? (Type uppercase yes): YES
	#	Enter passphrase for /dev/nvme0n1p2:
	#	Verify passphrase:
Add a corresponding entry to crypttab(5) with mapped device name nvme0n1p2_crypt, open it afterwards and check it’s contents manually.
 
	echo "nvme0n1p2_crypt UUID=$(sudo blkid -o value -s UUID /dev/nvme0n1p2) none luks" | sudo tee -a /etc/crypttab
	sudo cryptdisks_start nvme0n1p2_crypt
	#	Starting crypto disk...nvme0n1p2_crypt (starting)...
	#	Please unlock disk nvme0n1p2_crypt:  ********
	#	nvme0n1p2_crypt (started)...done.
Create a file system on the mapped device. Assuming source device for /boot is specified by its UUID in the fstab(5) – which the Debian Installer does by default – reusing the old UUID avoids editing the file.
 
	sudo grep /boot /etc/fstab
	#	/boot was on /dev/nvme0n1p2 during installation
	#	UUID=<uuid> /boot           ext4    defaults        0       2
	 
	sudo mkfs.ext4 -m0 -U <uuid> /dev/mapper/nvme0n1p2_crypt
	#	abcdef 1.23.4 (15-Dec-2018)
	#	Creating filesystem with 246784 1k blocks and 61752 inodes
	#	Filesystem UUID: x-x-x-x-x
	#	[…]
Finally, mount /boot again from fstab, and copy the saved tarball to the freshly encrypted file system.
 
	sudo mount -v /boot
	#	mount: /dev/mapper/nvme0n1p2_crypt mounted on /boot.
 
	sudo tar -C /boot --acls --xattrs -xf /tmp/boot.tar
	sudo mount -v /boot/efi
# Enabling cryptomount in GRUB2 + REMOVABLE MODE, KEYSLOT SETUP & AUTO-UNLOCK
Enable the feature, update the GRUB image and reinstall in removable mode:
 
	echo "GRUB_ENABLE_CRYPTODISK=y" | sudo tee -a /etc/default/grub
	sudo update-grub
 
	sudo grub-install /dev/nvme0n1 --force-extra-removable --no-nvram --uefi-secure-boot
	sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --boot-directory=/boot --bootloader-id=GRUB --force-extra-removable --no-nvram --uefi-secure-boot
 
	sudo cryptsetup luksDump /dev/nvme0n1p2 | grep -B1 "Iterations:"
	#	Key Slot 0: ENABLED
	#	Iterations:             1000000
 
	sudo cryptsetup luksChangeKey --pbkdf-force-iterations 420690 /dev/nvme0n1p2
	# 	Enter passphrase to be changed:
	#	Enter new passphrase:
	#	Verify passphrase:
(You can reuse the existing passphrase in the above prompts.)

	sudo cryptsetup luksOpen --test-passphrase --verbose /dev/nvme0n1p2
	#	Enter passphrase for /dev/nvme0n1p2:
	#	Key slot 1 unlocked.
	#	Command successful.
# Avoiding the Extra Password Prompt
Generate the shared secret (here with 512 bits of entropy as it’s also the size of the volume key) inside a new file.

	sudo mkdir -m0700 /etc/keys
	bash
	( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/boot.key conv=excl,fsync )
	( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/swap.key conv=excl,fsync )
	( umask 0077 && sudo dd if=/dev/urandom bs=1 count=64 of=/etc/keys/root.key conv=excl,fsync )
	zsh
	sudo chmod u=rx,go-rwx /etc/keys
	sudo chmod u=r,go-rwx /etc/keys/boot.key
	sudo chmod u=r,go-rwx /etc/keys/swap.key
	sudo chmod u=r,go-rwx /etc/keys/root.key
Create new key slots with new key files.

	sudo cryptsetup luksAddKey /dev/nvme0n1p2 /etc/keys/boot.key
and add in the other voleumes too.

	sudo cryptsetup luksAddKey /dev/nvme0n1p3 /etc/keys/swap.key
	sudo cryptsetup luksAddKey /dev/nvme0n1p4 /etc/keys/root.key
add these entries into the ccrypttab:

	sudo sed -i "/^nvme0n1p2_crypt/c\nvme0n1p2_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p2) /etc/keys/boot.key luks,key-slot=1" /etc/crypttab
	sudo sed -i "/^nvme0n1p3_crypt/c\nvme0n1p3_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3) /etc/keys/swap.key luks,swap,discard,key-slot=1" /etc/crypttab
	sudo sed -i "/^nvme0n1p4_crypt/c\nvme0n1p4_crypt UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p4) /etc/keys/root.key luks,discard,key-slot=1" /etc/crypttab
make sure /etc/fstab is correct and comment out <#> the original entries with:

	sudo nano /etc/crypttab - the mapper name for boot may change, so you may want to run lsblk to check it.
	#
	#
	#
# Finishing up BTRFS
Now we can add the final modifications to our /etc/fstab for our btrfs filesystem with:

	sudo nano /etc/fstab
like so:
 
	/dev/mapper/nvme0n1p4_crypt	/		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@		0	0
	/dev/mapper/nvme0n1p4_crypt	/.snapshots     btrfs   defaults,noatime,ssd,compress=lzo,subvol=@.snapshots	0	4
	# /boot was on /dev/nvme0n1p2 during installation
	UUID=<uuid>	/boot	ext4	defaults,noatime				0	1
	# /boot/efi was on /dev/nvme0n1p1 during installation
	UUID=<uuid>			/boot/efi	vfat	umask=0077						0	1
	/dev/mapper/nvme0n1p4_crypt	/home		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@home		0	2
	/dev/mapper/nvme0n1p4_crypt	/root		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@root		0	3
	/dev/mapper/nvme0n1p4_crypt	/srv		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@srv		0	0
	/dev/mapper/nvme0n1p4_crypt	/tmp		btrfs	defaults,noatime,ssd,compress=lzo,subvol=@tmp		0	0
	/dev/mapper/nvme0n1p4_crypt	/usr/local	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@usr@local	0	0
	/dev/mapper/nvme0n1p4_crypt	/var/log	btrfs	defaults,noatime,ssd,compress=lzo,subvol=@var@log	0	0
	/dev/mapper/nvme0n1p3_crypt	none		swap	sw							0	0
	# /var/lib/gdm3*lightdm was on /dev/mapper/nvme0n1p4_crypt during installation
	UUID=<uuid>	/var/lib/gdm3*lightdm	btrfs	defaults,subvol=@var@lib@gdm3		0	0
	# /var/lib/AccountsService was on /dev/mapper/nvme0n1p4_crypt during installation
	UUID=<uuid>	/var/lib/AccountsService	btrfs	defaults,subvol=@var@lib@AccountsService	0	0 
when done editing save your work, exit the editor then run:

	sudo systemctl daemon-reload
finally lets tell initramfs to use our keyfile:

	echo "KEYFILE_PATTERN=/etc/keys/*.key" | sudo tee -a /etc/cryptsetup-initramfs/conf-hook
include restrictive permissions to avoid leaking key material:

	echo "UMASK=0077" | sudo tee -a /etc/initramfs-tools/initramfs.conf
regenerate initramfs:

	sudo update-initramfs -u -k all
update grub
	
	sudo update-grub
and reboot for the changes to take affect:

	sudo reboot
# We're in....

Finally, open the 'disks' gui via the application finder, select the boot partition and change it's password; first to something simple (like kali) and then back to the more secure one. once that change is complete, navigate to 'edit encryption options' and check the 'user options' radio button, then modify the password field to yours, check 'unlock at boot' and delete the 'nofail' option.
 
At this point I would open *snapper-gui*, make a new pre-boot configuration for root and drop the number of snapshots down to between 3 and 10 so as not to run out of space, unless you want to manually prune these as time goes on.

The extra space we allocated initially should make sure that any updates won't fill up the boot partition past the allotted space, ensuring that you can use this system for quite a long time!

By the way, thanks for following along!
 

before you go...

While learning and self-improvement may seem harmless, there are individuals who oppose such growth, often out of fear that their own flaws might be exposed.
 
Ethics and morality are often subjective. While I value forgiveness, I've also come to realize that the world isn't black and white. We must stand together, especially when feeling lost or scared, and strive for open communication in a world where our leaders spread seperatism within the populace.
 
Our society is entangled in red tape, often out of fear of disrupting the status quo. This repression can lead individuals to seek out 'quick fixes,' be it in the form of oil or cheap labor. It's a form of self-sabotage that stands in the way of collective well-being.

    ~h1-|t3k/n0|-l1f3~

If you found value in this guide and wish to support me, consider buying me a coffee or helping in any small way you can. Your support would be greatly appreciated, especially during challenging times.
 
    Buy Me a Coffee:
    	https://www.buymeacoffee.com/h1t3kd
    For crypto donations:
        Bitcoin: 36aqR7oXP4tLxqe1oHckXtdVEizSNaWxJX
	Message me for other addresses :)
