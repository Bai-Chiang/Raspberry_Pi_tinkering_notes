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

