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

```

```
# fdisk /dev/sdb
```

- Press `p` to print the partition table. It should be blank.
- Press `n` to create a new partition. The options you should choose are:
    - 
    -
- Press `p` to print the partition table again.
- Press `w` to write the partition table and quit.

Create the Raid 1 array. Note that we use the two **partitions** (eg `/dev/sdb1`) to create the raid **device** `/dev/md0`. If you choose the **device** `/dev/sdb` strange things will happen, eg, your raid array disappearing on reboot... (Probably.)

```
# mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1
# cat /proc/mdstat


# cat /proc/partitions

```

## Create the new filesystem, and copy existing data

Create the filesystem on the raid device.

```
# mkfs.ext4 /dev/md0
```

Create a mountpoint for the raid filesystem, and mount it.

```
# mkdir /mnt/md
# mount /dev/md0 /mnt/md
```

(You should probably wait for it to sync at this point.)

Copy existing files. This step might not be relevant, it depends if you have an existing directory which you wish to move to the new raid fs. In my case, I wish to move `/home` and `/var/lib/gitea`.

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

If you *didn't* already reboot, you might change `md127` to `md0`. This command appends the auto-creation information to `mdadm.conf`. Now edit this file. You should find it added a line like this:

```
ARRAY /dev/md0 metadata=1.2 name=devbox:0 UUID=XXXXXXXX:XXXXXXXX:XXXXXXX:XXXXXXXX
```

Check this now works by doing a reboot. You should see the raid re-assembled on reboot.

```
# cat /proc/mdstat
```

## Mount (bind) the new raid-backed filesystem "over" the existing directory

This step might seem a bit strange. What it does is mount a directory from the raid-backed filesystem "on top of" an existing directory. What this does is "mask" the exisitng directory (and all its contents) - replacing it with the directory from the raid filesystem. This step can be revered using `umount`.

From the user point of view, the existing directory works as before, but the contents can be different. We will see an example of how this works later.

```
# mount --bind /mnt/md/home /home
# mount --bind /mnt/md/var/lib/gitea /var/lib/gitea
```

### Example of what it does

