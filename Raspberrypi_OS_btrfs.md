# Raspberry Pi btrfs setup

[btrfs](https://wiki.archlinux.org/title/Btrfs) provides some very nice features like [snapshots](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots) backup and [transparent compression](https://wiki.archlinux.org/title/Btrfs#Compression) which could increase lifespan of SD card by reducing write data and may improve performance where IO bottle neck is more common when using Raspberry Pi 4 with SD card.

- The official Raspberry Pi OS uses ext4 as default root filesystem.




___
# Raspberry Pi OS

## Prepare btrfs SD card

- Download and flash official Raspberry Pi OS to an USB drive and boot up Raspberry Pi from the USB.
  (don't need micro SD card addapter when copying the system to the micro SD card.)
  Login as default user `pi` with password `raspberry`.
- Update system and install package `btrfs-progs`
  ```
  sudo apt update && sudo apt full-upgrade 
  sudo apt install btrfs-progs
  reboot
  ```
- Login pi, insert mirco SD card, for example its device name is `/dev/mmcblk0`.

- Create GPT partition
  ```
  # parted /dev/mmcblk0
  (parted) mklabel gpt
  ```
  The patition scheme follows [this](https://wiki.archlinux.org/title/Parted#UEFI/GPT_examples) example,
  with 512MiB boot partition (also EFI partition), 12GiB swap partition and remaining space as root partition.
  ```
  (parted) mkpart "EFI system partition" fat32 0% 512MiB
  (parted) set 1 esp on
  (parted) mkpart "swap partition" linux-swap 512MiB 12.5GiB
  (parted) mkpart "root partition" btrfs 12.5GiB 100%
  (parted) quit
  ```
- Format the EFI partiton
  ```
  # mkfs.fat -F32 /dev/mmcblk0p1
  ```
- Create swap partition
  ```
  # mkswap /dev/mmcblk0p2
  ```
- Create btrfs filesystem and subvolumes
  ```
  # mkfs.btrfs /dev/mmcblk0p3
  # mount /dev/mmcblk0p3 /mnt
  # btrfs subvolume create /mnt/@
  # btrfs subvolume create /mnt/@home
  # btrfs subvolume create /mnt/@snapshots
  # mkdir /mnt/@/home
  # mkdir /mnt/@/boot
  # mkdir /mnt/@/.snapshots
  # umount -R /mnt
  ```
  
- Mount filesystem
  ```
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/mmcblk0p3 /mnt
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/mmcblk0p3 /mnt/home
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/mmcblk0p3 /mnt/.snapshots
  # mount /dev/mmcblk0p1 /mnt/boot
  ```
- system clone with [rsync](https://wiki.archlinux.org/title/Rsync#Full_system_backup).

  **No trailing slash at the end of `/mnt`**
  ```
  # rsync -aAXHv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /mnt
  ```

- Edit `/mnt/boot/cmdline.txt` 
  ```
  root=UUID=xxxxxxx-xxxx rootflags=subvol=/@ rootfstype=btrfs ... fsck.repair=no ...
  ```
  where `xxxxxxx-xxxx` is the `UUID` of root patition, you can get it using `lsblk -dno UUID /dev/mmcblk0p3`.
  Add kernel parameter `rootflags=subvol=/@`.
  Change `rootfstype=ext4` to `rootfstype=btrfs`,and
  set `fsck.repair` to `no`, see [`fsck.btrfs(8)`](https://man.archlinux.org/man/fsck.btrfs.8).

- Edit `/mnt/etc/fstab`
  ```
  UUID=BOOT_UUID  /boot        vfat   defaults                                                                     0  0
  UUID=ROOT_UUID  /            btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@	         0  0
  UUID=ROOT_UUID  /home        btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home	     0  0
  UUID=ROOT_UUID  /.snapshots  btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@snapshots	 0  0
  UUID=SWAP_UUID  none         swap   defaults                                                                     0  0
  ```
  Note that the last tow collums should be `0` not `1`, see [this](https://wiki.archlinux.org/title/Fstab#Usage)
  Get `BOOT_UUID` from `lsblk -dno UUID /dev/mmcblk0p1`, and `lsblk -dno UUID /dev/mmcblk0p2` for `SWAP_UUID`, and `lsblk -dno UUID /dev/mmcblk0p3` for `ROOT_UUID`.

- poweroff
  ```
  # umount -R /mnt
  # poweroff
  ```

## Cross-compile Linux Kernel
This section assume building 64-bit kernel for raspberry pi 4, for other cases read the [official document](https://www.raspberrypi.org/documentation/computers/linux_kernel.html#cross-compiling-the-kernel) and make cooresponding changes.

- Set up cross-compile environment following the first part of [this](https://github.com/Bai-Qiang/Raspberry_Pi_tinkering_notes/blob/main/Cross_compile_Linux_kernel.md#create-a-clean-debian-environment) guide
  on your PC.
- Plug in the SD card to your PC. Using `lsblk` find out its label, for example `/dev/sdX1` for boot partition and `/dev/sdX2` for root partition.
- Boot up the debian container
  ```
  sudo systemd-nspawn -bD ~/debian-systemd-nspawn --bind=/dev/sdX1 --bind=/dev/sdX2
  ```
- Clone the raspberry pi linux kernel source
  ```
  cd ~
  git clone --depth=1 https://github.com/raspberrypi/linux
  ```
- Apply default configuration
  ```
  cd linux
  KERNEL=kernel8
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
  ```
- Config the kernel using `menuconfig`
  ```
  make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
  ```
- Active Btrfs in kernel
  ```
  File systems  --->
      <*> Btrfs filesystem support
  ```
  Make sure it's build into the kernel `<*>` not as a module `<M>`.
- Active initramfs support, for `root=UUID=xxx` in `cmdline.txt`
  ```
  File systems  --->
      <*> Btrfs filesystem support
  ```
- Save and exit.
- Rebuild kernel
  ```
  make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
  ```
  `-jN` where I choose `N` as the number of logical cores which can be obtained from `nproc` command.
  For example, `-j12` for a 6 core 12 threads processor.

- Mount the SD card inside the container as follows
  ```
  mkdir /mnt/boot
  mkdir /mnt/root
  mount /dev/sdX1 /mnt/boot
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/sdX2 /mnt/root
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/sdX2 /mnt/root/home
  ```
- Install kernel module to the SD card
  ```
  env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/mnt/root modules_install
  ```
- Copy the kernel and Device Tree to the SD card
  ```
  cp arch/arm64/boot/Image /mnt/boot/$KERNEL-btrfs.img
  cp arch/arm64/boot/dts/broadcom/*.dtb /mnt/boot/
  cp arch/arm64/boot/dts/overlays/*.dtb* /mnt/boot/overlays/
  cp arch/arm64/boot/dts/overlays/README /mnt/boot/overlays/
  ```
- Edit the `/mnt/boot/config.txt` file let raspberry pi boot into new kernel
  ```
  kernel=kernel8-btrfs.img
  ```
  replace `kernel8-btrfs.img` with `$KERNEL-btrfs.img` in the previous step.
  You could run `echo $KERNEL-btrfs.img` to get its name.
- Umount the SD card
  ```
  umount -R /mnt/boot
  umount -R /mnt/root
  ```
- Shut down the container
  ```
  poweroff
  ```

___
- Insert the SD card in to raspberry pi and it should be able to boot from `btrfs` root filesystem.
- Disable swapfile
  ```
  sudo dphys-swapfile swapoff
  sudo systemctl disable dphys-swapfile.service
  ```
  If swapfile is needed following [this](https://wiki.archlinux.org/title/Btrfs#Swap_file) guide.



