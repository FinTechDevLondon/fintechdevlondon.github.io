---
layout: post
title: Setting up mdadm raid partitions on Debian Linux
---

## Setting up mdadm raid partitions on Debian Linux

I have written this guide to document the steps required to introduce raid partitions to a Debian Linux system after installation. The commands used are quite generic across Linux systems, so this should work on all flavors of Linux with minimal modification.

### Situation

I recently deployed a (physical) server to my home network running Debian 11 for the purposes of installing and testing Gitea (Self-hosted Github alternative). Now that I am satisfied Gitea is working correctly, and that I will be using it for some extended period of time, I want to put the data it stores on a software raid parition. This guide explains how that is done.

This guide explains *how to create a raid system, and how to move non-raid directories which exist in the filesystem to a raid-backed filesystem*.

In this guide we will

- Install mdadm, and create a Raid 1 (mirror) array
- Setup partitions, fstab and mount pointes


## Installing `mdadm`, and prerequisites

I have a computer with 3 SSDs.

- A smaller SSD which contains the operating system, and all my files, including the `/home` directory. This drive is not new, and it has a significant amount of wear.
- 2 1TB SSDs, which are new, and blank.

I have installed Debian 11 on the smaller disk. Since this disk has a significant amount of wear, it is probably not wise to use it to store important files, such as my git repos. Hence, we will move important files to a new, raid, filesystem. Although I don't expect Gitea / git to be writing a large amount of data to my filesystem, I intend to use this as a central place to host all my personal work, so taking the risk just isn't worth it.

I also installed Gitea - but this isn't a prerequisite, and is optional. You can use your system for whatever you want, obviously.

The only things we need to install which didn't come installed by default are `mdadm` and `rsync`.

```
$ su - -l
# apt update && apt install mdadm rsync
```

## Setup the Partitions and Raid Array

First we create new partitions on our blank drives. My disks are `/dev/sdb` and `/dev/sdc` - yours may be different, so check first.

```
# cat /proc/partitions
major minor  #blocks  name

   8       16  976762584 sdb
   8       32  976762584 sdc
   8        0  234431064 sda
   8        1   19529728 sda1
   8        2  214899712 sda2
```

Create the partitions with `fdisk`.

```
# fdisk /dev/sdb
```

- Press `p` to print the partition table. It should be blank.
- Press `n` to create a new partition. The options you should choose are:
    - `p` for primary partition
    - `1` for first partition
    - Accept the defaults for everything else
- Press `p` to print the partition table again.
- Press `w` to write the partition table and quit.

Create the Raid 1 array. Note that we use the two **partitions** (eg `/dev/sdb1`) to create the raid **device** `/dev/md0`. If you choose the **device** `/dev/sdb` strange things will happen, eg, your raid array disappearing on reboot... (Probably.)

```
# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
# cat /proc/mdstat
```

```
cat /proc/partitions
major minor  #blocks  name

   8       16  976762584 sdb
   8       17  976761560 sdb1
   8       32  976762584 sdc
   8       33  976761560 sdc1
   8        0  234431064 sda
   8        1   19529728 sda1
   8        2  214899712 sda2
   9        0  976629440 md0
```

## Create the new filesystem, and copy existing data

Create the filesystem on the raid device. As an asside it is useful to know that a filesystem is fundamentally different to a partition - in case you did not already realize. If you usually use a graphical interface such as `gparted` (or similar) for partition management, the distinction between the two concepts is usually less obvious.

```
# mkfs.ext4 /dev/md0
```

Create a mountpoint for the raid filesystem, and mount it.

```
# mkdir /mnt/md
# mount /dev/md0 /mnt/md
```

At this point, while you can technically proceed and begin using the raid array, I noticed some slightly unusual behaviour when rebooting before the array has finished its initial sync. Therefore you should probably wait for it to sync at this point. There is an initial sync process which is required to ensure the data on the disks is mirrored. This occurs even if the disks are blank, although it is possible to skip this check.

While the disk is syncing, the output of `cat /proc/mdstat` will show something similar to the following.

```
# cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
md0 : active raid1 sdb1[0] sdc1[1]
      976629440 blocks super 1.2 [2/2] [UU]
      [===>.................]  resync = 16.9% (166020544/976629440) finish=181.4min speed=74464K/sec
      bitmap: 8/8 pages [32KB], 65536KB chunk
```

Next, copy any existing files. This step might not be relevant to all usescases. It depends if you have an existing directory which you wish to move to the new raid fs. In my case, I wish to move `/home` and `/var/lib/gitea`, so I need to copy these directories.

```
# rsync -av /home /mnt/md/
# mkdir -p /mnt/md/var/lib
# rsync -av /var/lib/gitea /mnt/md/var/lib
```

## Setup the raid array for auto-reassembly on reboot

If you reboot the system now, you will either find there is no raid system on reboot, or it looks a bit weird. This is what my system looked like after a reboot.

```
# cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
md127 : active (auto-read-only) raid1 sdc1[1] sdb1[0]
      976629440 blocks super 1.2 [2/2] [UU]
        resync=PENDING
      bitmap: 7/8 pages [28KB], 65536KB chunk

unused devices: <none>
```

As you can see, a device `md127` has been created automatically. We would have expected it to be `md0`. If you look at the contents of the file `/etc/mdadm/mdadm.conf` you will find it doesn't contain much information.

Run the following command.

```
mdadm --detail --scan /dev/md127 >> /etc/mdadm/mdadm.conf
```

If you *didn't* already reboot, you might need to change `md127` to `md0` when running the above scan command. This command appends the auto-creation information to `mdadm.conf`. Now edit this file. You should find it added a line like this:

```
ARRAY /dev/md127 metadata=1.2 name=devbox:0 UUID=XXXXXXXX:XXXXXXXX:XXXXXXX:XXXXXXXX
```

You may need to change it to be like this:

```
ARRAY /dev/md0 metadata=1.2 UUID=XXXXXXXX:XXXXXXXX:XXXXXXX:XXXXXXXX
```

You may also need to run

```
# update-initramfs -u
```

Check the raid array now comes up after a reboot with the expected number by issuing a reboot.

```
# cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
md0 : active (auto-read-only) raid1 sdc1[1] sdb1[0]
      976629440 blocks super 1.2 [2/2] [UU]
      bitmap: 0/8 pages [0KB], 65536KB chunk
```

## Mount (bind) the new raid-backed filesystem "over" the existing directory

This step might seem a bit strange. What it does is mount a directory from the raid-backed filesystem "on top of" an existing directory. What this does is "mask" the exisitng directory (and all its contents) - replacing it with the directory from the raid filesystem. This step can be revered using `umount`.

From the user point of view, the existing directory works as before, but the contents can be different.

```
# mount --bind /mnt/md0/home /home
# mount --bind /mnt/md0/var/lib/gitea /var/lib/gitea
```

To explain this another way, the command `mount --bind /mnt/md0/home /home` "replaces" `/home` with the contents of `/mnt/md0/home`. No files are lost in this process, the old directory and its constents still exist, and they will reappear if `/home` is unmounted again. With the new directory mounted on top of the existing one, the new contents can be used in the exact same way that the old directory contents could. Just keep in mind that if the raid array fails to start up or mount that the old directory might "reappear" giving the illusion that data has been lost. You can view the current mount scheme with the `mount` command.

## Setup `fstab` to mount the raid array at boot

Get the UUID of the raid array.

```
# blkid /dev/md0
```

Edit the file `/etc/fstab`. It should (probably) look something like this, but your specific usecase will vary.

```
UUID=a4a837b8-1bc3-4cfb-8f33-30d2fef4ad36 /               ext4    errors=remount-ro 0       1
UUID=daee4230-bff7-4fd2-be7b-0cf51d7b270d none            swap    sw              0       0
# RAID Devices
UUID=69e51a35-257c-46ea-abe5-7e1604d2d6f4 /mnt/md0        ext4    errors=remount-ro 0       0
# Mount the bind mountpoints
/mnt/md0/home /home                       none            defaults,bind           0       0
/mnt/md0/var/lib/gitea /var/lib/gitea     none            defaults,bind           0       0
```

## Wrap up

That's it. You should now be able to create RAID arrays, and swap out existing (non raid backed) directories for raid backed ones.
