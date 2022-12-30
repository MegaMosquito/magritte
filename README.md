# magritte

[Ceci n'est pas une repo.](https://en.wikipedia.org/wiki/The_Treachery_of_Images) Or is it?

There really isn't any actual code here. Instead, this is just a set of how-to recipes to help me manage disk images, mount file systems, and other similar tasks that I often need to do when working with small linux computers. In my opinion these tasks are not as easily accomplished in Linux (not even in Linux Desktop versions) as they are on MacOS and Windows. So for me, I need these recipes to get these jobs done in Linux. Maybe they'll help you too.

Recipes included here:

1. [How to create a ".img" file, and shrink it, and compress it.](https://github.com/MegaMosquito/magritte/blob/main/README.md#how-to-create-a-img-file-and-shrink-it-and-compress-it)

2. [How to permanently mount the file system from USB media.](https://github.com/MegaMosquito/magritte/blob/main/README.md#how-to-permanently-mount-the-file-system-from-usb-media)

3. [How to format a disk for Linux use.](https://github.com/MegaMosquito/magritte/blob/main/README.md#how-to-format-a-disk-for-linux-use)

## The Recipes

### How to create a ".img" file, and shrink it, and compress it.

 * Note: this technique uses tool uses my fork of [Drewsif/PiShrink](https://github.com/Drewsif/PiShrink). I forked it to keep a stable version under my control. You could use the original instead.

This recipe shows how to create an image **file** from a MicroSD disk. I sometimes build a useful MicroSD for my Raspberry Pi machines and then use this recipe to create an image file from it so I can burn that image onto more MicroSD disks for other machines.

NOTE: You will need significantly more free disk space than the size of your entire MicroSD source disk on the machine where you are using to create this disk image. I have been building my source images on 8GB MicroSD cards and using a 32GB disk on the Pi that I use to burn the images. You might be able to get away with 16GB source images, but for my purposes 8GB has always been more than enough.

Here's the procedure I use:

**1.**  Before attaching the disk, run `lsblk` to find the existing diskS

**2.**  Physically attach the disk (e.g., using a USB MicroSD reader)

**3.**  Run `lsblk` again to find the path to this disk. Below shows typical output, where the disk labelled `sda` here is an 8GB MicroSD disk with a Raspberry Pi OS image on it (where `sda` means `/dev/sda` is the path to the disk):

```
 $ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    1  7.5G  0 disk 
├─sda1        8:1    1  256M  0 part /media/pi/boot
└─sda2        8:2    1  7.3G  0 part /media/pi/rootfs
mmcblk0     179:0    0 29.7G  0 disk 
├─mmcblk0p1 179:1    0  256M  0 part /boot
└─mmcblk0p2 179:2    0 29.5G  0 part /
 $ 
```

**4.**  Use `dd` to copy the entire disk into a file. This file will be the size of the disk. The `dd` command has this form: `dd bs=1M if=INPUT of=OUTPUT` where `INPUT` will be the disk you are copying from and `OUTPUT` will be the file where you want it to be written. Here's an example command and it's output for the disk shown above (it will take several minutes to run):

```
 $ sudo dd bs=1M if=/dev/sda of=/home/pi/image.img
7680+0 records in
7680+0 records out
8053063680 bytes (8.1 GB, 7.5 GiB) copied, 543.281 s, 14.8 MB/s
 $ 
```

**5.** Now you can remove the MicroSD card since it is no longer needed

**6.** Shrink and then compress the image. Shrinking removes all of the unused blocks from the disk image, very significantly reducing its size. This tool also configures the disk image so it will be automatically expanded to the full size of whatever MicroSD you burn it onto. For example, this 8GB image file will shrink to about 3GB, and then I will burn it onto a 32GB MicroSD and the file system will automatically resize on first boot so the entire 32GB is available. To shrink, I use [https://github.com/MegaMosquito/PiShrink](https://github.com/MegaMosquito/PiShrink). So clone this repo, and then use the shell script it contains to shrink the image. Here's an example command (image shrinking is very fast but if you use the`-z` argument to gzip it like I did below, then it will again take several minutes to compress the image file):

```
 $ sudo ./pishrink.sh -vzp image.img
...
pishrink.sh: Shrunk image.img.gz from 7.5G to 1.1G ...
 $ 
```

The "shrunken" image (before compression) was about 3GB:
```
-rw-r--r--  1 root root 3122004480 Oct 15 15:22 image.img
```

**7.**  At this point I would give file ownership to the pi user:

```
 $ sudo chown pi:pi image.img.gz
```


### How to permanently mount the file system from USB media.

Sometimes I want to mount a large disk drive onto one of my small Linux computers. For example, when I create a Network Addressable Storage (NAS) service or a Media Server (like Plex). I usually use a Linux **Desktop** distribution in these cases, and they will automatically mount the drive when you do that. However, they create temporary mounts. The recipe here will show you how to mount a disk if it is not mounted, and then it will show you how to make a disk mount permanent, such that Linux will mount it each time the machine boots.

Here's the procedure I use:

**1.**  Physically connect the disk. For my Raspberry Pi machines this is always done using the USB connectors.

**2.**  Once the disk is connected, Linux will assign it a name under `/dev`. You now need to figure out that name. You can use `lsblk` for this, as the **root** user. For example:

```
 $ sudo lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0  1.4T  0 disk 
└─sda1        8:2    0  1.4T  0 part 
mmcblk0     179:0    0 29.7G  0 disk 
├─mmcblk0p1 179:1    0  256M  0 part /boot
└─mmcblk0p2 179:2    0 29.5G  0 part /
 $ 
```

This shows the boot disk of your machine (mounted at `/boot`) and the root of your Linux file system, mounted at `/`. It asl shows the USB disk has been given device address `/dev/sda` and the main partition of this disk (of size 1.4TB in the example above) is given the address `/dev/sda1`. This partition is not currently mounted as you can tell because nothing appears in the **MOUNTPOINT** column in the listing.

If only a **disk** shows up under **TYPE**, and not a **part* then you may need to format this disk before using it. There's a recipe for that below. Go ahead and format it then start over on this recipe.

Now let's manually mount it.

**3.**  Create a mount point for the disk partition, then mount it.

Begin by creating a place in the `/` file system for the disk to be mounted. In Raspberry Pi OS disks owned by the **pi** user are normally mounted under `/media/pi`. So let's create the mount point named "**my-disk**" there:

```
 $ mkdir /media/pi/my-disk
```

Note that a mount point directory like this must exist before you use the `mount` command to mount your disk **partition** there. I emphasized partition there because you aren't mounting the disk, but instead you are mounting the partition. Partitions appear in the `lsblk` output with **TYPE** "**part**". In the above example, the disk is `/dev/sda` but the partition is `/dev/sda1`. Let's mount the partition now:

```
 $ mount /dev/sda1 /media/pi/my-disk
```

Verify that the mount has been completed, by running the `mount` command without any arguments to list all mounts. You could also run the `sudo lsblk` command again to see the mountpoint for the partition.

Note that after the partition is mounted, its files are directories can be viewed in the file system using normal commands, or using the visual file browser on your Desktop. You can `cd` to the mount point, and `'ls`, etc.

Now let's make this mount permanent. We will need some more information about the partition first.

**4.** Get the partition's details

To make the mount permanent, you first need to determine the partition's **UUID** and file system **TYPE**. We will use the `blkid` command with the partition's device path (`/dev/sda1` in the example above) to get those details:

```
 $ blkid /dev/sda1
/dev/sda1: LABEL_FATBOOT="EFI" LABEL="EFI" UUID="67E3-17ED" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="ccfbbbe6-9fe1-4840-beb6-587a24cf648e"
 $ 
```

In this example, the partition's UUID is "ccfbbbe6-9fe1-4840-beb6-587a24cf648e" and the file system TYPE is "vfat".

Note that in my experience, Linux does not handle non-Linux file systems well. So my advice is to always use Linux-formatted **ext4** type file systems for uses like NAS or Media Servers. These ext4 Linux file systems can still be shared using CIFS/Samba but since Linux is hosting the files it works best when it is a Linux file system. In theory other file system types can work, but I have had bad experiences so I do not recommend using anything but Linux-formatted disks. Maybe the issues I had have been fixed, but for me it's not worth the risk. If you wish to format the disk as **ext4** see the disk formatting recipe below and then restart this recipe.

**5.**  Permanently mount the disk

**NOTE**: To make the mount permanent, you must edit the system `/etc/fstab` file. This is a key component of your Linux system. If you make an error in editing this file you may render your system unbootable. It is strongly recommended that you make a complete system backup before you begin, in case you need to revert to that backup. There is a recipe above that will enable you to create a snapshot of an entire disk in an "**.img**" file. You could do that and save the file somewhere else (off this machine) before you begin. It is also wise to make a copy of this file before you make changes. Then you most likely can rescue an unbootable system by mounting it on another Linux host and restoring this file from your original copy.

To begin, take a look at your `/etc/fstab` file, and create a copy of the original. E.g.:

```
:~ $ cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=4df93e91-01  /boot           vfat    defaults          0       2
PARTUUID=4df93e91-02  /               ext4    defaults,noatime  0       1
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
:~ $ cp /etc/fstab /etc/fstab.original
:~ $ 
```

Add a line with this form:

```
    <UUID> <MOUNTPOINT> <TYPE> defaults 0 2
```

You can Google for more details on `/etc/fstab` but the safest approach is to always add ` defaults 0 2`. The last number should always be 2. The second to last should always be 0. `defaults` is the safest default value for the third to last field. Fields are separated by one or more tabs or spaces.

For example, I modified the file above to mount a Linux disk by adding one line:

```
:~ $ cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=4df93e91-01  /boot           vfat    defaults          0       2
PARTUUID=4df93e91-02  /               ext4    defaults,noatime  0       1
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
UUID=c40a7586-641e-48d6-b9fe-342fabcccbe5 /media/pi/PLEXDATA ext4 defaults 0 2
:~ $ 
```

After making the changes, unmount the disk manually, then see if it will re-mount using your changes to the permanent mounts. (the `mount -a` command tries to mount all of the permanent mounts that are not currently mounted). Oh, and please note that the command to unmount partitions in Linux is `umount` (not unmount, as one might expect).

```
 $ umount /media/pi/my-disk
 $ mount -a
```

Then check that it is remounted by running `mount` with no arguments or `sudo lsblk`. If the mount point shows up in that output, try exploring the files below the mount point. If this all looks good, then reboot the machine, and check again after rebooting.

### How to format a disk for Linux use.

Formatting a disk for Linux, from the Linux CLI is relatively easy with this recipe:

**1.**  Physically attach the disk

**2.**  Find the device path of the disk. You can use `sudo lsblk` for this:

```
 $ sudo lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 238.5G  0 disk 
└─sda1        8:1    0 238.5G  0 part /media/pi/PLEXDATA
mmcblk0     179:0    0  28.9G  0 disk 
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0  28.6G  0 part /
 $ 
```

Make sure you know which disk you want to format, because formatting it will remove all references to anything that was previously on this disk.

**3.**  Use a CLI command to unmount the disk if it is mounted (but leave it attached):

Note that the command to unmount partitions in Linux is `umount` (not unmount, as one might expect).

```
 $ umount /media/pi/PLEXDATA
```

Then run the `sudo lsblk` command again to verify the **MOUNTPOINT** has been removed.

**4.**  Partition and format the disk

Note that in my experience, Linux does not handle non-Linux file systems well. So my advice is to always use Linux-formatted **ext4** type file systems for disks used on Linux systems.  In theory other file system types can work, but I have had bad experiences so I do not recommend using anything but Linux-formatted disks. Maybe those issues have been fixed but I won't risk it.

The command to partition and format a disk in Linux is `mkfs`. You just need to provide the device path of the target disk (`/dev/sda1` in the example below) and the type of formatting you want to apply (**ext4** is used below and I recommend this for any disk you plan to use on a Linux system).

**NOTE** Please be careful when specifying the disk device path because this command will wipe the target disk.

```
sudo mkfs -t ext4 /dev/sda1
```

When the formatting completes you can mount the disk as described in another recipe above and then start using it.


