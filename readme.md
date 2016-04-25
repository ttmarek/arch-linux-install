# INSTALLATION STEPS

## References
  * [Arch Linux, Beginner's Guide](https://wiki.archlinux.org/index.php/Beginners%27_guide)

## Assumptions
  * You're installing Arch on a machine with a UEFI motherboard.
  * You're using the GPT partition table type.
  * You don't want to have any swap space (paid good money for RAM)
  * You want to keep `/` root and `/home` in the same partition
  * You are installing arch on an intel-cpu


## Preparation
If you're running Arch in VirtualBox make sure you give the virtual
machine at least 256 MB of RAM and 800 MB of disk space.

Go
[here](https://wiki.archlinux.org/index.php/Category:Getting_and_installing_Arch)
for instructions on downloading the installation iso and methods for
booting it on different machines.

### Connect to the internet
You shouldn't have to do anything if you've got a wired
connection. But its best to make sure you have a connection by pinging
google:

```
root@archiso ~ # ping www.google.com
```

Press `Control-C` to stop the ping. If you've got an internet
connection you should see messages like these:

```
PING www.google.com (24.244.4.119) 56(84) bytes of data.
64 bytes from 24.244.4.119 (24.244.4.119): icmp_seq=1 ttl=63 time=22.6ms
64 bytes from 24.244.4.119 (24.244.4.119): icmp_seq=2 ttl=63 time=14.0ms
```

### Update the system clock

```
root@archiso ~ # timedatectl set-ntp true
root@archiso ~ # timedatectl status
```

Check to make sure the UTC time displayed by the last command is accurate.

## Prepare the storage devices

### Useful commands for this section

List all devices connected to your system:

```
root@archiso ~ # lsblk
```

Devices (e.g. hard disks) will be listed as `sdx`, where `x` is a
lower-case letter starting from `a` for the first device (`sda`), `b`
for the second device (`sdb`), and so on. Partitions on those devices
will be listed as `sdxY`, where `Y` is a number starting from `1` for
the first partition, `2` for the second, and so on.

Get information (e.g. file system type, partition table type) on a
specific device:

```
root@archiso ~ # parted /dev/sda print
```

### Create a new partition table

Open the drive you want to install Arch in using `parted` (e.g `sda`):

```
root@archiso ~ # parted /dev/sda
```

Create a new GPT partition table (UEFI systems):

```
(parted) mklabel gpt
```

### Create partitions and map them to directories

#### Rules
    1. A partition must be created and mapped to the `/` root
       directory
    2. When using a UEFI motherboard, one EFI System Partition must be
       created.
    3. Partitions cannot overlap each other. If you do not want to
      leave unused space in the device, make sure that each partition
      starts where the previous one ends.

#### Create the partitions

Create a special bootable EFI System Partition. Its recommend to make
this partition 513MiB.

```
(parted) mkpart ESP fat32 1MiB 513MiB
```

See what you made (make note of the partition number as it'll be used
in the next command):

```
(parted) print
```

Flag the EFI partition as bootable (`1` is the EFI partition number):

```
(parted) set 1 boot on
```

Ensure that the partition has `boot, esp` flags (last column):

```
(parted) print
```

Create a partition for swap space (e.g. `~500MiB`):

```
(parted) mkpart primary linux-swap 513MiB 1GiB
```

Create a partition for the remaining space (`/` and `/home` will go here):

```
(parted) mkpart primary ext4 1GiB 100%
```

View your handywork (make note of the partition numbers):

```
(parted) print
```
#### Setup and activate swap

Exit out of `parted`, then setup and activate the swap partition you
made. The following example assumes the swap partition is on device
`a` and the swap partition number is `2` (`sda2`). You can check this
by either running `print` in `parted` or by running `lsblk`.

```
(parted) quit
root@archiso ~ # mkswap /dev/sda2
root@archiso ~ # swapon /dev/sda2
```

When you run `lsblk` now you should see `[SWAP]` under `MOUNTPOINT`
for the swap partition.

#### Format the file systems

Now you have to format the remaining partitions with the appropriate
file system.

Format the EFI partition as a `fat32` file system. The following
example assumes the EFI partition is on device `a` and that the
partition number is `1` (`sda1`).

```
root@archiso ~ # mkfs.fat -F32 /dev/sda1
```

Format the remaining partition (the one for `/` and `/home`) as `ext4`
file systems:

```
root@archiso ~ # mkfs.ext4 /dev/sda3
```

Run `parted /dev/sda print` to check to make sure each partition is
formatted correctly.

#### Map the partitions to directories in the live system (create mount points)

Mount the EFI partition (e.g `sda1`) to `/mnt/boot`:

```
root@archiso ~ # mkdir /mnt/boot
root@archiso ~ # mount /dev/sda1 /mnt/boot
```

Mount the root partition (e.g `sda3`) to `/mnt`:

```
root@archiso ~ # mount /dev/sda3 /mnt
```

### Install the base packages

```
root@archiso ~ # pacstrap -i /mnt base base-devel
```

### Configuration

Generate an [fstab](https://wiki.archlinux.org/index.php/Fstab) file:

```
root@archiso ~ # genfstab -U /mnt >> /mnt/etc/fstab
```

Chroot to the new system in `/mnt`:

```
root@archiso ~ # arch-chroot /mnt /bin/bash
```

Set the system locale. Open up `/etc/locale.gen` in `nano` and uncomment
`en_US.UTF-8 UTF-8` and save the file. Then, run:

```
[root@archiso /]# locale-gen
Generating locales...
  en_US.UTF-8... done
Generation complete.
```

Open up `/etc/locale.conf` in `nano` (it won't exist yet), then add in
a line like this one `LANG=en_US.UTF-8`. Here you're setting one of the
locale's you generated previously as the system language. When setting
`LANG` only use the first column of one of the languages you
uncommented in `/etc/locale.gen`.

Select a time zone:

```
[root@archiso /]# tzselect
```

Create a symbolic link `/etc/localtime`, where `Zone/Subzone` is the `TZ` value from tzselect:

```
[root@archiso /]# ln -s /usr/share/zoneinfo/Zone/SubZone /etc/localtime
```

E.g. for America/Edmonton you enter `ln -s /usr/share/zoneinfo/America/Edmonton /etc/localtime`

It is recommended to adjust the time skew, and set the time standard to UTC:

```
[root@archiso /]# hwclock --systohc --utc
```

### Install a boot loader
