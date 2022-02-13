# Arch Linux ARM on CM4 and boot from nvme ssd

- [change boot order](https://github.com/Bai-Chiang/Raspberry_Pi_tinkering_notes/blob/main/CM4_NVME_boot.md)


- Flash a micro SD with official Raspberry pi OS image, and boot into it with nvme ssd in the m.2 slot.

- All following command needs run as root user
  ```
  sudo -i
  ```

- Install packages
  ```
  apt install btrfs-progs libarchive-tools
  ```

- partition the nvme device
  ```
  fdisk /dev/nvme0n1
  ````

  - Type o. Create new DOS partition table.
  - Type n -> p -> 1 -> ENTER -> +512MiB. Create a 512MiB partition at front.
  - Type t -> c . Set the first partition type to W95 FAT32 (LBA).
  - Type n -> p -> 2 -> ENTER -> ENTER. Create the second partition for the remaining space.
  - Type p. Check partition size and Types are correct.
  - Type w. Write the partition table and exit.

- Create FAT filesystem for /boot
  ```
  mkfs.vfat /dev/nvme0n1p1
  ```
- Create btrfs filesystem and subvolumes
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

- Mount btrfs filesystem
  ```
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/nvme0n1p2 /mnt
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/nvme0n1p2 /mnt/home
  mount -o ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
  mount /dev/nvme0n1p1 /mnt/boot
  ```
- Download and extract the root filesystem as **root** not via sudo.
  ```
  wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz
  bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C /mnt
  sync
  ```

- update `fstab`
  ```
  $EDITOR /mnt/etc/fstab
  ```
  The `fstab` file should looks like this
  ```
  # <file system>                            <dir>        <type>  <options>                                                                     <dump>  <pass>
  UUID=xxxx-xxxx                             /boot        vfat    defaults                                                                      0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /            btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@            0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /home        btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home        0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /.snapshots  btrfs   rw,ssd,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@snapshots   0       0
  ```

  note that the last tow collums should be `0` not `1`, see [this](https://wiki.archlinux.org/title/Fstab#Usage).
  You can get UUID for `/dev/sdX1` by running this command `lsblk -dno UUID /dev/nvme0n1p1`, same for `/dev/nvme0n1p2`.

- Edit `/mnt/boot/boot.txt`, change root UUID 
  ```
  ... root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx rootflags=subvol=/@ ...
  ```
- Edit/Add other configs modification to `/mnt/boot/config.txt`, for example to enable USB interface on the CM4 add 
  `dtoverlay=dwc2,dr_mode=host` to `config.txt`.
  
- Unmount the SD card
  ```
  umount -R /mnt
  ```

