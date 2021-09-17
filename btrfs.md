# Raspberry Pi btrfs setup (In progress)


The official Raspberry Pi OS uses ext4 as default root filesystem.
[btrfs](https://wiki.archlinux.org/title/Btrfs) provides some very nice features like [snapshots](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots) backup and [transparent compression](https://wiki.archlinux.org/title/Btrfs#Compression) which could increase lifespan of SD card by reducing write data and may improve performance where IO bottle neck is more common when using Raspberry Pi 4b with SD card.

___
## Install Raspberry Pi OS

- Download and flash official Raspberry Pi OS to an SD card and boot up Raspberry Pi. Login as default user `pi` with password `raspberry`.
- Update system and install package `btrfs-progs`
  ````
  sudo apt update && sudo apt full-upgrade 
  sudo apt install btrfs-progs
  reboot
  ```
- Login pi, Insert new SD card via usb adaptor, for example its device name is `/dev/sda`.

- Run following commands as root user
    ```
    sudo -i
    ```

- Unmount SD card
    ```
    umount /dev/sda*

    ```

- partition the SD card
    ```
    fdisk /dev/sda
    ````

    - Type o. Create new DOS partition table.
    - Type n -> p -> 1 -> ENTER -> +512MiB. Create a 512MiB partition at front.
    - Type t -> c . Set the first partition type to W95 FAT32 (LBA).
    - Type n -> p -> 2 -> ENTER -> ENTER. Create the second partition for the remaining space.
    - Type p. Check partition size and Types are correct.
    - Type w. Write the partition table and exit.

- Create and mount FAT filesystem
    ```
    mkfs.vfat /dev/sda1
    mkdir -p /mnt/boot
    mount /dev/sda1 /mnt/boot
    ```
- Create btrfs filesystem and subvolumes
    ```
    mkfs.btrfs /dev/sda2
    mkdir /mnt/root
    mount /dev/sda2 /mnt/root
    btrfs subvolume create /mnt/root/@
    btrfs subvolume create /mnt/root/@home
    btrfs subvolume create /mnt/root/@snapshots
    mkdir /mnt/root/@/home
    mkdir /mnt/root/@/.snapshots
    umount -R /mnt/root
    ```

- Mount btrfs filesystem
    ```
    mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/sda2 /mnt/root
    mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/sda2 /mnt/root/home
    mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/sda2 /mnt/root/.snapshots
    ```
- system clone with [rsync](https://wiki.archlinux.org/title/Rsync#Full_system_backup). **There is trailing slash "/" at the end of  `/boot/` but no trailing slash at the end of `/mnt/boot` and `/mnt/root`**
    ```
    rsync -aAXHv /boot/ /mnt/boot
    rsync -aAXHv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/boot/*"} / /mnt/root
    ```

- Get new root partition PARTUUID
    ```
    lsblk -dno PARTUUID /dev/sda2
    ```

- Edit `/mnt/boot/cmdline.txt` 
    ```
    ... root=PARTUUID=xxxxxxxx-02 rootfstype=btrfs ... fsck.repair=no
    ```
    where `xxxxxxxx-02` is the `PARTUUID` you get in previous step. Change `rootfstype=ext4` to `rootfstype=btrfs`,and
    set `fsck.repair` to `no`.

- Edit `/mnt/root/etc/fstab`
    ```
    ...
    PARTUUID=xxxxxxxx-01  /boot        vfat   defaults 0 0
    PARTUUID=xxxxxxxx-02  /            btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@	         0 0
    PARTUUID=xxxxxxxx-02  /home        btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home	     0 0
    PARTUUID=xxxxxxxx-02  /.snapshots  btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@snapshots	 0 0
    ...
    ```
    note that the last tow collums should be `0` not `1`.

- Unmount SD card and poweroff
    ```
    umount -R /mnt/boot
    umount -R /mnt/root
    poweroff
    ```

## Cross-compile Linux Kernel

- Set up cross-compile environment following the first part of [this](https://github.com/Bai-Qiang/Raspberry_Pi_tinkering_notes/blob/main/Cross_compile_Linux_kernel.md#create-a-clean-debian-environment) guide
  on your PC.
- Plug in the SD card fomatted as `btrfs` to your PC. Using `lsblk` find out its label, for example `/dev/sda1` for boot partition and `/dev/sda2` for root partition.
- Boot up the debian container
    ```
    sudo systemd-nspawn -bD ~/debian-systemd-nspawn --bind=/dev/sda1 --bind=/dev/sda2
    ```
  


