# LUKS encrypted root partition

Here's my personal guide for encrypting your existing root partition and making it boot again.

***WARNING:*** Encrypting a single partition is not great for the same reason having the entire system installed on the same partition is not great. I personally use btrfs so I made an exception to this rule since btrfs supports subvolumes and subvolume quotas.

This guide uses an arbitrary number of encrypted partitions, you can encrypt everything, **EXCEPT**: the partition in which resides grub (so the boot partition) and the efi partition. If you don't have a separate partition for boot, sorry but this guide will not work. There are methods to move the boot data elsewhere but I won't discuss it in this guide.

With this being said this guide will also support multiple encrypted partitions with differen encryption keys, so no problem if your system is installed on multiple partitions.

## -1. Make a backup for god's sake
Self explanatory

## 0. Use a safe environment
To do all these steps is better to use a live iso of arch or any other operating system because we cannot operate on partitions that are mounted.

## 1. Encrypt you partitions
First thing to do is actually creating this (or these) encrypted partitions. You have two options:

### Option 1.1
**luksFormat**:
Destroy everything on your current partition and creating a new filesystem in which placing the new os/files.
1. Start by going into fdisk and creating a new partition (if you have already a partition skip to step 2):
    - `fdisk /dev/sdX` (where X is the letter, no numbers)
        - if you have and nvme device it'll be probably something like `/dev/nvme0n1` the `0` is the disk number and the `1` is the namespace. If you are not sure use `fdisk -l` and find your disk.
        - The logic is not to use files that end in `...pX` with X a number, because this specific files are partitions.
    - `n` (to create a new partition)
    - specify partition start and end
2. Then we format this said partition:
**I will stress it further: THIS WILL DELETE ALL THE DATA IN THIS PARTITION, if the partition is empty, nice, if it's not, you will lose it.**
`cryptsetup luksFormat /dev/sdXY` (where X is the device letter and Y the partition number.)
    - Again if on nvme it will be something like `nvme0n1p4`

Ok, now we have a luks partition! Go to the next step and then go to `Mount these bloody partitions`
    
### Option 1.2
**reencrypt**:
in-place conversion from plain data to encrypted partition (for this method, a backup is very highly recommended). But for doing this we will need to leave some space for the luks header, so we will need to resise the partition target by say 50MB. The header will take 32MB or so, when this process is finished we can regrow the partition to make it occupy all the available space.

So let's start by shrinking the actual partition based on the filesystem you have:
    
- **btrfs**:
very simple: you don't even have to unmount your disk. If you have unmounted it then remount it, then use this command:
`btrfs filesystem resize {newsize} {mountpoint}`
    - *newsize*: can be absolute size like `100G`, so the partition will be 100 GB big, or it can be relative, so `-100M` will reduce the partition be 100 MB.
    - *mountpoint*: where the partition is mounted (or a subfolder of said mountpoint). eg: `/mnt/data`

  If this fails, it probably mean your disk is full.
- **ext4/3/2**:
It's more complicated but only by one step: determine the actual size, because `resize2fs` does not support relative sizes like btrfs.
    - Finding the partition size in bloks:
    Give a `df` and find your target partition and copy the number under the header `1K-bloks` and save it to a mental variable named `partition_blocks`.
    - Umount the partition
    `umount /dev/{partition}`
    - Check the partition before shrink:
    `e2fsck /dev/{partition}`
	- Edit the mental variable `partition_blocks` to a value you like
	*eg: you have a 100GB partition so `partition_blocks` will be 1048576, and we want to shrink it by 50MB, then we subtract 50MB worth of 1K blocks (`50MB = 51200KB`) from our variable: 
	```partition_blocks = 104857600 - 51200 = 104806400```*
    - Shrink:
    `resize2fs /dev/{partition} {partition_blocks}K` ← the K is **very** important!
    
Then we can proceed to encrypt the partition:
`cryptsetup reencrypt --encrypt --reduce-device-size 32M /dev/{partition}`
This will ask you to confirm and will say that your disk is about to get wiped, but don't worry this is not the case. Your disk will be completely rewritten (henche the long time taken for this operation), but your data is still there, encrypted, but there.
After your confirmation it'll ask for a passphrase, you will be able to change it later but don't forget it or you will lose your data!

When this finishes we can test our results:
- Open the luks partition:
`cryptsetup open /dev/{partition} {mapping-name}`
    - *mapping-name*: arbitrary, it can be whatever you like.
- Mount the mapping:  
*cryptsetup creates a virtual device named `/dev/mapper/{mapping-name}`, that when accessed is dynamically decrypted on read and dynamically encrypted on write by the `dm-crypt` linux subsystem.*
`mount /dev/mapper/{mapping-name} {mountpoint}`
    - *mapping-name*: the one you chose before.
    - *mountpoint*: wherever you like, I suggest under the `/mnt` folder.
    
If all your data is there then good work! The conversion was successfull! If the conversion has corrupted the partition in some way, the kernel will probably give some errors when mounting.

## Mounting these bloody partitions
Now we have a partition that does not work by itself: we need to open the mapping (if you converted an already existing partition you already have when we tested the new encryption).
- Open the mapping:
`cryptsetup open /dev/{partition} {mapper-name}`
- Mount partition
	- **ext4** and other:
		`mount /dev/mapper/{mapper-name} {mountpoint}`
	- **btrfs** if you have multiple subvolumes:
		`mount /dev/mapper/{mapper-name} -o subvol={subvol} {mountpoint}`

For the next steps we need to `chroot` inside our system so mount every partition needed for making your system work (eg. `/`, `/home`, `/boot`, `/boot/efi`).
Use the steps above to mount all the needed partition, and then use `chroot {newroot}` to enter your subsystem. If you use arch (btw), then use `arch-chroot {newroot}`.

## Getting grub and initramfs to work
*Everything in this and the next steps will need to be done inside the chroot environment*

### Early Hooks
This method uses initramfs hooks that starts systemd in an earlier stage compared to a normal system and ask us a password for each disk we want to be mounted early on.
To do this we need to configure our system to properly generate our initram image, for making sure it includes `systemd` as an early hook.
To do this we need to:
- Edit this file: `/etc/mkinitcpio.conf`
    in particular changing the line that looks like this:
    `HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)`
    to
    `HOOKS=(base systemd autodetect keyboard sd-console modconf block sd-encrypt filesystems keyboard fsck)`
- One subtle step (in which I personally struggled) was the `vconsole.conf` file: during the recreation of the initial ramdisk when you include `sd-encrypt` mkinitcpio will search inside the file `/etc/vconsole.conf` to find which keyboard layout you are using, if it doesn't it will fail to insert `sd-encrypt` silently (aka without us knowing), so it's important to make sure this file exists and inside it there is this line:
`KEYMAP={keymap}` where *keymap* is your keyboard layout (eg. `us`, `it`, ecc. Use `localectl list-keymaps` for finding the right one for you.)
- Next we create this file `/etc/crypttab.initramfs` and inside of it we insert the disk we want to unlock:
	```text
	# Mount /dev/sdaX as /dev/mapper/root, /dev/mapper/home and /dev/mapper/var using LUKS
	#  It will ask us for the passphrase at boot time.
	
	# Example:
	# mapper-name  /dev/{partition}   {passphrase file or mode}   {options}
	root           /dev/sda3          tpm2-device=auto            ses
	home           /dev/sda4                                                  # specifying nothing will prompt for password.
	var            /dev/sda5          /home/alice/var_passphrase.key
	```
	*`/etc/crypttab.initramfs`  will be included in the initial ramdisk image under `/etc/crypttab` and is used by `sd-encrypt` to know which partitions, disks or virtual partitions will need to be unlocked before booting. For more information on this file formatting look at the [this page](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#crypttab) on the arch wiki.*
- Now we have to update our `/etc/fstab`: if we only used UUID then we won't have any issues, because the partition uuid is mantained during encryption and therefore after the mapping has been open, the partition uuid will encrypted along the data. If we instead used device number then we will need to modify the lines...
	```text
	# ...from this
	/dev/sda1			/	ext4	defaults,noatime	0 0
	
	# to this
	/dev/mapper/{mapper-name}	/	ext4	defaults,noatime	0 0
	# or use partitions uuids,
	# I personally prefer those since they don't change.
	```
- Then we have to regenerate our initial ramdisk, on archlinux you should be able to do so only by reinstalling the kernel, so `pacman -S linux [linux-zen, ...]` on arch/manjaro or `apt install linux [linux-zen, ...]` on debian based.
### Grub
The second step is telling our kernel that our root partition is not normal, and must be unencrypted first. 
To do so we need to:
- Edit the line in `/etc/default/grub` that looks like this:
`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"`
Add the following string `root=/dev/mapper/{mapping-name}`, so that it will look something like this:
`GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet root=/dev/mapper/root"`
This will ensure that the system uses our mapping and not our physical disk during the boot process .
- Regenerate the grub configuration:
`grub-mkconfig -o /boot/grub/grub.cfg`

## Try
Exit the chroot environment and reboot, try if it works!

## Sources:
- ArchLinux Wiki:
	- https://wiki.archlinux.org/title/dm-crypt/Device_encryption
	- https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition
	- https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Using_systemd-cryptsetup-generator
	- https://wiki.archlinux.org/title/Dm-crypt/System_configuration#crypttab
	- https://wiki.archlinux.org/title/Linux_console/Keyboard_configuration
