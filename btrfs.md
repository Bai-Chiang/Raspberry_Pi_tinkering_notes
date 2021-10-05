# Raspberry Pi btrfs setup

[btrfs](https://wiki.archlinux.org/title/Btrfs) provides some very nice features like [snapshots](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Snapshots) backup and [transparent compression](https://wiki.archlinux.org/title/Btrfs#Compression) which could increase lifespan of SD card by reducing write data and may improve performance where IO bottle neck is more common when using Raspberry Pi 4 with SD card.

- The official Raspberry Pi OS uses ext4 as default root filesystem.

- The [`manjaro-arm-installer`](https://gitlab.manjaro.org/manjaro-arm/applications/manjaro-arm-installer/-/tree/master) script provides `btrfs` option, and during the installation process you can choose different desktop environments or window managers.



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

- Run following commands as root user
  ```
  sudo -i
  ```

- Unmount SD card (because if using raspberry pi OS desktop version it would auto-mount USB dirve)
  ```
  umount /dev/mmcblk0*
  ```

- partition the SD card
  ```
  fdisk /dev/mmcblk0
  ````

  - Type o. Create new DOS partition table.
  - Type n -> p -> 1 -> ENTER -> +512MiB. Create a 512MiB partition at front.
  - Type t -> c . Set the first partition type to W95 FAT32 (LBA).
  - Type n -> p -> 2 -> ENTER -> ENTER. Create the second partition for the remaining space.
  - Type p. Check partition size and Types are correct.
  - Type w. Write the partition table and exit.

- Create and mount FAT filesystem
  ```
  mkfs.vfat /dev/mmcblk0p1
  mkdir -p /mnt/boot
  mount /dev/mmcblk0p1 /mnt/boot
  ```
- Create btrfs filesystem and subvolumes
  ```
  mkfs.btrfs /dev/mmcblk0p2
  mkdir /mnt/root
  mount /dev/mmcblk0p2 /mnt/root
  btrfs subvolume create /mnt/root/@
  btrfs subvolume create /mnt/root/@home
  mkdir /mnt/root/@/home
  umount -R /mnt/root
  ```

- Mount btrfs filesystem
  ```
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/mmcblk0p2 /mnt/root
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/mmcblk0p2 /mnt/root/home
  ```
- system clone with [rsync](https://wiki.archlinux.org/title/Rsync#Full_system_backup).
  **There is trailing slash "/" at the end of  `/boot/` but no trailing slash at the end of `/mnt/boot` and `/mnt/root`**
  ```
  rsync -aAXHv /boot/ /mnt/boot
  rsync -aAXHv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/boot/*"} / /mnt/root
  ```

- Get new root partition PARTUUID
  ```
  lsblk -dno PARTUUID /dev/mmcblk0p2
  ```

- Edit `/mnt/boot/cmdline.txt` 
  ```
  root=PARTUUID=xxxxx-02 rootflags=subvol=/@ rootfstype=btrfs ... fsck.repair=no ...
  ```
  where `xxxxx` is the `PAATUUID` you get in previous step. Change `rootfstype=ext4` to `rootfstype=btrfs`,and
  set `fsck.repair` to `no`, see [`fsck.btrfs(8)`](https://man.archlinux.org/man/fsck.btrfs.8).

- Edit `/mnt/root/etc/fstab`
  ```
  ...
  UUID=xxxxx  /boot        vfat   defaults 0 0
  UUID=xxxxx  /            btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@	         0 0
  UUID=xxxxx  /home        btrfs  rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home	     0 0
  ...
  ```
  Note that the last tow collums should be `0` not `1`, see [this](https://wiki.archlinux.org/title/Fstab#Usage)
  For `/boot` partition, get UUID from `lsblk -dno UUID /dev/mmcblk0p1`,
  root partition use `lsblk -dno UUID /dev/mmcblk0p2`.

- Unmount SD card and poweroff
  ```
  umount -R /mnt/boot
  umount -R /mnt/root
  poweroff
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



___
# Arch Linux ARM
(Arch Linux ARM does not boot from my CM4)

This part follows arch linux arm [installation guide](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4).
Suppose the SD card device shown as `/dev/sdX`.

- All following command needs run as root user
  ```
  sudo -i
  ```

- partition the SD card
  ```
  fdisk /dev/sdX
  ````

  - Type o. Create new DOS partition table.
  - Type n -> p -> 1 -> ENTER -> +512MiB. Create a 512MiB partition at front.
  - Type t -> c . Set the first partition type to W95 FAT32 (LBA).
  - Type n -> p -> 2 -> ENTER -> ENTER. Create the second partition for the remaining space.
  - Type p. Check partition size and Types are correct.
  - Type w. Write the partition table and exit.

- Create and mount FAT filesystem
  ```
  mkfs.vfat /dev/sdX1
  mkdir -p /mnt/boot
  mount /dev/sdX1 /mnt/boot
  ```
- Create btrfs filesystem and subvolumes
  ```
  mkfs.btrfs /dev/sdX2
  mkdir /mnt/root
  mount /dev/sdX2 /mnt/root
  btrfs subvolume create /mnt/root/@
  btrfs subvolume create /mnt/root/@home
  btrfs subvolume create /mnt/root/@snapshots
  mkdir /mnt/root/@/home
  mkdir /mnt/root/@/.snapshots
  umount -R /mnt/root
  ```

- Mount btrfs filesystem
  ```
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/sdX2 /mnt/root
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/sdX2 /mnt/root/home
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/sdX2 /mnt/root/.snapshots
  ```
- Download and extract the root filesystem as **root** not via sudo.
  ```
  wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz
  bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C /mnt/root
  sync
  ```
- Move boot files to boot partition
  ```
  mv /mnt/root/boot/* /mnt/boot
  ```
- update `fstab`
  ```
  $EDITOR /mnt/root/etc/fstab
  ```
  The `fstab` file should looks like this
  ```
  # <file system>                            <dir>        <type>  <options>                                                                     <dump>  <pass>
  UUID=xxxx-xxxx                             /boot        vfat    defaults                                                                      0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /            btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@            0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /home        btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home        0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /.snapshots  btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@.snapshots  0       0
  ```

  note that the last tow collums should be `0` not `1`, see [this](https://wiki.archlinux.org/title/Fstab#Usage).
  You can get UUID for `/dev/sdX1` by running this command `lsblk -dno UUID /dev/sdX1`, same for `/dev/sdX2`.

- Edit `/mnt/boot/boot.txt`, change root UUID 
  ```
  ... root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rootflags=subvol=/@ ...
  ```
- Edit/Add other configs modification to `/mnt/boot/config.txt`, for example to enable USB interface on the CM4 add 
  `dtoverlay=dwc2,dr_mode=host` to `config.txt`.
- Unmount the SD card
  ```
  umount -R /mnt/boot
  umount -R /mnt/root
  ```

