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
  # mkfs.fat -F32 /dev/nvme0n1p1
  ```
- Create swap partition
  ```
  # mkswap /dev/nvme0n1p2
  ```
- Create btrfs filesystem and subvolumes
  ```
  # mkfs.btrfs /dev/nvme0n1p3
  # mount /dev/nvme0n1p3 /mnt
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
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/nvme0n1p3 /mnt
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/nvme0n1p3 /mnt/home
  # mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/nvme0n1p3 /mnt/.snapshots
  # mount /dev/nvme0n1p1 /mnt/boot
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
  where `xxxxxxx-xxxx` is the `UUID` of root patition, you can get it using `lsblk -dno UUID /dev/nvme0n1p3`.
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
  Get `BOOT_UUID` from `lsblk -dno UUID /dev/nvme0n1p1`, and `lsblk -dno UUID /dev/nvme0n1p2` for `SWAP_UUID`, and `lsblk -dno UUID /dev/nvme0n1p3` for `ROOT_UUID`.

- restart 
  ```
  # umount -R /mnt
  # poweroff
  ```
  remove SD card, then poweron raspberry pi. 


