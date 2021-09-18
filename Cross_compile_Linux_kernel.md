# Raspberry Pi cross-compiling Linux kernel

[The official document](https://www.raspberrypi.org/documentation/computers/linux_kernel.html#cross-compiling-the-kernel)
recommend using a debian virtual machine to cross-compiling Raspberry Pi Linux kernel.
But virtual machine is not as powerful as physical machine.
In this document I will use [systemd-nspawn](https://wiki.archlinux.org/title/Systemd-nspawn), a chroot on steroids, to create a clean debian environment,
and compiling linux kernel inside this container.
This note assume building 64-bit kernel for raspberry pi 4, for other cases read the [official document](https://www.raspberrypi.org/documentation/computers/linux_kernel.html#cross-compiling-the-kernel) and make cooresponding changes.

___

## [Create a clean debian environment](https://wiki.archlinux.org/title/Systemd-nspawn#Create_a_Debian_or_Ubuntu_environment)
- Install [debootstrap](https://archlinux.org/packages/?name=debootstrap) and [debian-archive-keyring](https://archlinux.org/packages/?name=debian-archive-keyring)
    ```
    sudo pacman -S debootstrap debian-archive-keyring
    ```
- [Set up debian environment](https://wiki.archlinux.org/title/Systemd-nspawn#Create_a_Debian_or_Ubuntu_environment)
    ```
    sudo debootstrap --include=systemd-container --components=main,universe stable ~/debian-systemd-nspawn https://deb.debian.org/debian
    ```
    Here I create a container at my home directory `~` called `debian-systemd-nspawn`.
    The repository-url is`https://deb.debian.org/debian` __without a trailing slash__.
- Set up root password. Use systemd-nspawn chroot into new environment
    ```
    sudo systemd-nspawn -D ~/debian-systemd-nspawn
    ```
    now you are inside a debian environment
    ```
    passwd
    ```
    type
    ```
    logout
    ```
    close the container.
    
- Boot up the container
    ```
    sudo systemd-nspawn -bD ~/debian-systemd-nspawn 
    ```
    the `-b` option tell nspawn boot up the container. `-D` tells systemd-nspawn the directory.
- Now you boot up the container. Login as `root` user, and your root password. 
- Full system update
    ```
    apt update && apt full-upgrade
    ```
    
    If the terminal in the container could't recognize `backspace` or `tab` key, run
    ```
    echo 'export TERM=xterm-256color' >> ~/.bashrc
    ```
    in the container, and re-login.  
    
- [Install dependencies and 64-bit toolchain](https://www.raspberrypi.org/documentation/computers/linux_kernel.html#install-required-dependencies-and-toolchain)
     ```
    apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev crossbuild-essential-arm64
    ```
    
- Stop the container by runing command
    ```
    poweroff
    ```
    inside your container.
    If you want to force terminate your container, hold `Ctrl` and rapidly press `]`.

___
## Configure and Recompile Kernel

- Plug in your SD card (already has Raspberry Pi OS installed on it) and using `lsblk` find out its label, here let's say the two partitions on the SD card are `/dev/sda1` for root and `/dev/sda2` for boot.

- Star the container with this command
    ```
    sudo systemd-nspawn -bD ~/debian-systemd-nspawn --bind=/dev/sda1 --bind=/dev/sda2
    ```
    where `--bind=` option let the container access directory `/dev/sda1` and `/dev/sda2`.
    
- [Get kernel source](https://www.raspberrypi.org/documentation/computers/linux_kernel.html#get-the-kernel-sources)
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

- Config kernel using `menuconfig`
    ```
    make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
    ```
- Exit and save the changes
- Build kernel
    ```
    make -j12 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
    ```
    `-jN` where I choose `N` as the number of logical cores which can be obtained from `nproc` command.
    For example, `-j12` for a 6 core 12 threads processor.
    (Using `systemd-nspawn` all CPU cores are available, unlike virtual machine it's not recommended to allowcate all host cpu cores to guest machine.)

- Mount the SD card inside the container as follows
    ```
    mkdir /mnt/boot
    mkdir /mnt/root
    mount /dev/sda1 /mnt/boot
    mount /dev/sda2 /mnt/root
    ```
- Install kernel module to the SD card
    ```
    env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=/mnt/root modules_install
    ```
- Copy the kernel and Device Tree to the SD card
    ```
    cp arch/arm64/boot/Image /mnt/boot/$KERNEL-myconfig.img
    cp arch/arm64/boot/dts/broadcom/*.dtb /mnt/boot/
    cp arch/arm64/boot/dts/overlays/*.dtb* /mnt/boot/overlays/
    cp arch/arm64/boot/dts/overlays/README /mnt/boot/overlays/
    ```
- Edit the `/mnt/boot/config.txt` file let raspberry pi boot into new kernel
    ```
    kernel=kernel8-myconfig.img
    ```
    replace `kernel8-myconfig.img` with `$KERNEL-myconfig.img` in the previous step.
    You could run `echo $KERNEL-myconfig.img` to get its name.
- Umount the SD card
    ```
    umount -R /mnt
    ```
- Shut down the container
    ```
    poweroff
    ```
 - Insert the SD card in to raspberry pi it should boot with new kernel.
