- [update boot loader](https://www.raspberrypi.com/documentation/computers/compute-module.html#cm4bootloader) for CM4 on another raspberry pi 

- on another raspberry pi install build packages then download and compile `usbboot`
  ```
  sudo apt install git libusb-1.0-0-dev build-essential
  git clone --depth=1 https://github.com/raspberrypi/usbboot
  cd usbboot
  make
  ```
  
- Then modify CM4 bootloader
  ```
  cd recovery
  ```
  edit `boot.conf`, set
  ```
  BOOT_ORDER=0xf561
  ```
  This let CM4 first try to boot from SD card, then NVME,then USB 2.0. See [this](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-4-bootloader-configuration) for more options.
  
- update the EEPROM image and flash bootloader
  ```
  ./update-pieeprom.sh
  sudo ../rpiboot -d . 
  ```
  It will wait for CM4 connection. Now connect raspberry pi CM4 to PC (make sure not boot CM4).

- Now you can copy system to NVME ssd and boot from it.

  For graphical installation,
  using the `SD Card Copier` copy the current raspberry pi os to nvme ssd, check `New Partition UUIDs`.
  
  For Lite installation,  see below.
  
  - partition the nvme device
    ```
    # parted /dev/mmcblk0
    (parted) mklabel gpt
    ```
    The patition scheme has 512MiB boot partition (also EFI partition), and remaining space as root partition.
    ```
    (parted) mkpart "EFI partition" fat32 1MiB 513MiB
    (parted) set 1 esp on
    (parted) mkpart "root partition" ext4 513MiB 100%
    (parted) quit
    ```

  - Format disk
    ```
    sudo mkfs.fat -F32 /dev/nvme0n1p1
    sudo mkfs.ext4 /dev/nvme0n1p2
    ```
  - Mount disk
    ```
    mount /dev/nvme0n1p2 /mnt
    mkdir /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot
    ```
  - system clone with [rsync](https://wiki.archlinux.org/title/Rsync#Full_system_backup).

    **No trailing slash at the end of `/mnt`**
    ```
    # rsync -aAXHv --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} / /mnt
    ```
  - update root device in `/mnt/boot/cmdline.txt` 
    ```
    root=/dev/nvme0n1p2
    ```
    and `/mnt/etc/fstab`
    ```
    /dev/nvme0n1p1  /boot        vfat   defaults             0  0
    /dev/nvme0n1p2  /            ext4   defaults,noatime	   0  0
    ```

  - reboot
     ```
     umount -R /mnt
     poweroff
     ```
     remove SD card and power on, now you should boot into the SSD.



