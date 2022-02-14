# Raspberry Pi OS btrfs on nvme ssd

The official Raspberry Pi OS does not build btrfs into the kernel, it's build as a module.
So we need to build a initramfs.

- Boot into Raspberry Pi OS disable swapfile
  ```
  sudo dphys-swapfile swapoff
  sudo systemctl disable dphys-swapfile.service
  ```
  If swapfile is needed following [this](https://wiki.archlinux.org/title/Btrfs#Swap_file) guide.
  
- Prepare initramfs

  Install necessary packages
  ```
  # apt install btrfs-progs initramfs-tools
  ```
  add `btrfs` module to `/etc/initramfs-tools/modules`
  
  build initramfs
  ```
  # mkinitramfs -o /boot/initramfs-btrfs.gz
  ```
  add this line to `/boot/config.txt`
  ```
  initramfs initramfs-btrfs.gz
  ```
  then reboot raspberry pi, check everything work correctly.

- [set up nvme boot.](https://github.com/Bai-Chiang/Raspberry_Pi_tinkering_notes/blob/main/CM4_NVME_boot.md)

- partition and format the nvme device
  ```
  # parted /dev/nvme0n1
  (parted) mklabel gpt
  ```
  The patition scheme has 512MiB boot partition (also EFI partition), and remaining space as root partition.
  ```
  (parted) mkpart "EFI partition" fat32 1MiB 513MiB
  (parted) set 1 esp on
  (parted) mkpart "root partition" btrfs 513MiB 100%
  (parted) quit
  ```

  Create FAT filesystem for /boot
  ```
  mkfs.fat -F32 /dev/nvme0n1p1
  ```
  Create btrfs filesystem and subvolumes
  ```
  mkfs.btrfs /dev/nvme0n1p2
  mount /dev/nvme0n1p2 /mnt
  btrfs subvolume create /mnt/@
  btrfs subvolume create /mnt/@home
  btrfs subvolume create /mnt/@snapshots
  mkdir /mnt/@/home
  mkdir /mnt/@/boot
  mkdir /mnt/@/.snapshots
  umount -R /mnt
  ```

  Mount btrfs filesystem
  ```
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/nvme0n1p2 /mnt
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/nvme0n1p2 /mnt/home
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
  mount /dev/nvme0n1p1 /mnt/boot
  ```
- system clone with [rsync](https://wiki.archlinux.org/title/Rsync#Full_system_backup).

  **No trailing slash at the end of `/mnt`**
  ```
  # rsync -aAXHv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /mnt
  ```
- update root device in `/mnt/boot/cmdline.txt`  add `rootflags=subvol=/@`, change `rootfstype=ext4` and `fsck.repair=yes` to `rootfstype=btrfs` and `fsck.repair=no`
  ```
  ... root=/dev/nvme0n1p2 rootflags=subvol=/@ 
  ```
  and `/mnt/etc/fstab`
  ```
  /dev/nvme0n1p1  /boot        vfat   defaults             0  2
  /dev/nvme0n1p2  /            btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@            0       0
  /dev/nvme0n1p2  /home        btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home        0       0
  /dev/nvme0n1p2  /.snapshots  btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@snapshots   0       0
  ```

- unplug SD card and reboot
  ```
  umount -R /mnt
  poweroff
  ```
  remove SD card and power on, now you should boot into the SSD.


