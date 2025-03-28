---
layout: post
title: "HOWTO: Mount LUKS Btrfs for arch-chroot"
date: 2025-03-27
tags: luks archlinux btrfs arch-chroot
---

LUKS and BTRFS make for an interesting combination if you ever need to perform some work on another drive. This is a step-by-step procedure I wrote for myself and thought it would be useful to share with others.

The [ArchWiki](https://wiki.archlinux.org/title/Main_page) already covers how to do this if you combine a few separate articles, but I wanted a unified document.

## Mounting
1. Identify the drive to mount.
    ```bash
    $ sudo lsblk
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
    ...output trimmed...
    nvme1n1     259:0    0 931.5G  0 disk 
    ├─nvme1n1p1 259:1    0     1G  0 part 
    └─nvme1n1p2 259:2    0 930.5G  0 part

    $ export DRIVE="nvme1n1"
    ```
    I have trimmed the output to show only the drive applicable here. I am also exporting it as a variable to use later.

2. Decrpyt the drive, if encrypted, otherwise skip this step.
    ```bash
    $ sudo --preserve-env=DRIVE cryptsetup luksOpen /dev/"${DRIVE}p2" luksvol
    Enter passphrase for /dev/nvme1n1p2: 

    ```
    *Note: If the prompt appears to freeze, be patient. It takes a few seconds to decrypt.*

3. Mount the `@` subvolume.
    ```bash
    $ sudo mount --options subvol=@ /dev/mapper/luksvol /mnt
    ```
    The `@` subvolume is the `/` directory. It needs to be mounted first.

4. Mount the `@home` subvolume.
    ```bash
    $ sudo mount --options subvol=@home /dev/mapper/luksvol /mnt/home
    ```

5. Mount the `@pkg` subvolume.
    ```bash
    $ sudo mount --options subvol=@pkg /dev/mapper/luksvol /mnt/var/cache/pacman/pkg
    ```

6. Mount the `@log` subvolume.
    ```bash
    $ sudo mount --options subvol=@log /dev/mapper/luksvol /mnt/var/log
    ```

7. Mount the `@snapshots` subvolume.
    ```bash
    $ sudo mount --options subvol=@snapshots /dev/mapper/luksvol /mnt/.snapshots
    ```
    The following message may appear if this subvolume doesn't exist. You can safely proceed.
    ```
    mount: /mnt/.snapshots: fsconfig() failed: No such file or directory.
    ```

8. Mount the `boot` partition. This is critical if you want to update the kernel or bootloader.
    ```bash
    $ sudo --preserve-env=DRIVE mount /dev/"${DRIVE}p1" /mnt/boot
    ```

9. Finally you can `arch-chroot` into the mounted system.
    ```bash
    $ sudo arch-chroot /mnt
    ```

## Cleanup
After exiting the `arch-chroot`, unmount everything.
```bash
$ sudo umount --recursive /mnt
```

If the drive was encrypted, reencrypt.
```bash
$ sudo cryptsetup close luksvol
```

### References:
1. https://wiki.archlinux.org/title/Dm-crypt
2. https://wiki.archlinux.org/title/Chroot#Running_on_Btrfs
