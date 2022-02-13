# Raspberry Pi OS btrfs setup

The official Raspberry Pi OS does not build btrfs into the kernel, to being able to use btrfs as root filesystem we need to recompile Linux kernel.

This note assume building 64-bit kernel for raspberry pi 4, for other cases read the [official document](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#building) and make cooresponding changes.

If you build kernel on the raspberry pi locally, you can skip this section.

## Cross-compile environment
- Set up cross-compile environment following the first part of [this](https://github.com/Bai-Qiang/Raspberry_Pi_tinkering_notes/blob/main/Cross_compile_Linux_kernel.md#create-a-clean-debian-environment) guide
  on your PC.
- Plug in the SD card to your PC. Using `lsblk` find out its label, for example `/dev/sdX1` for boot partition and `/dev/sdX2` for root partition.
- Boot up the debian container
  ```
  sudo systemd-nspawn -bD ~/debian-systemd-nspawn --bind=/dev/sdX1 --bind=/dev/sdX2
  ```
  
If you build kernel locally, when running `make` don't need `ARCH=xxx`.
## Configure kernel
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
Boot into btrfs system and disable swapfile
  ```
  sudo dphys-swapfile swapoff
  sudo systemctl disable dphys-swapfile.service
  ```
  If swapfile is needed following [this](https://wiki.archlinux.org/title/Btrfs#Swap_file) guide.



