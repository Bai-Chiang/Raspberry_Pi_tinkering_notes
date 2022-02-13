## [update boot loader](https://www.raspberrypi.com/documentation/computers/compute-module.html#cm4bootloader) for CM4 on a ubuntu PC

- on PC install build packages then download and compile `usbboot`
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
  
- update the EEPROM image
  ```
  ./update-pieeprom.sh
  sudo ../rpiboot -d . 
  ```
  It will wait for CM4 connection. Now connect raspberry pi CM4 to PC (make sure not boot CM4).
  
- Flash bootloader
  ```
  cd ..
  sudo ./rpiboot -d recovery
  ```
  if it shows Waiting for BCM2835/6/7/2711... for a long time, you can unplug and replug CM4


## Copy system to nvme ssd

- Flash a Raspberry pi OS to SD card, or using existing SD card. Insert both SD card and nvme to CM4, and boot. 
  It should boot from the SD card, because we configured CM4 to try boot from SD card first.

- I will use btrfs on the ssd, so we need to install `btrfs-progs` package.

- Create GPT partition
  ```
  # parted /dev/nvme0n1
  (parted) mklabel gpt
  ```
  The patition scheme follows [this](https://wiki.archlinux.org/title/Parted#UEFI/GPT_examples) example,
  with 512MiB boot partition (also EFI partition), 8GiB swap partition and remaining space as root partition.
  ```
  (parted) mkpart "EFI system partition" fat32 0% 512MiB
  (parted) set 1 esp on
  (parted) mkpart "swap partition" linux-swap 512MiB 4.5GiB
  (parted) mkpart "root partition" btrfs 8.5GiB 100%
  (parted) quit
  ```
- Format the EFI partiton
  ```
  # mkfs.fat -F32 /dev/nvme0n1p1
  ```
- Create swap partition
  ```
  mkswap /dev/nvme0n1p2
  ```
