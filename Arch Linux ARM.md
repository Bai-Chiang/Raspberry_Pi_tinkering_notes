# Arch Linux ARM on CM4 and boot from nvme ssd

NOT working, see https://archlinuxarm.org/forum/viewtopic.php?f=65&t=15893&p=69001&hilit=btrfs#p69001

- [change boot order](https://github.com/Bai-Chiang/Raspberry_Pi_tinkering_notes/blob/main/CM4_NVME_boot.md)

- Flash a micro SD with official Raspberry pi OS image, and boot into the Raspberry OS on the SD card with nvme ssd pluged in the m.2 slot.

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

- Create FAT filesystem for /boot
  ```
  mkfs.fat -F32 /dev/nvme0n1p1
  ```
- Create btrfs filesystem and subvolumes
  ```
  mkfs.btrfs /dev/nvme0n1p2
  mount /dev/nvme0n1p2 /mnt
  btrfs subvolume create /mnt/@
  btrfs subvolume create /mnt/@home
  btrfs subvolume create /mnt/@snapshots
  btrfs subvolume create /mnt/@var_log
  btrfs subvolume create /mnt/@pacman_pkgs
  mkdir /mnt/@/{boot,home,.snapshots}
  mkdir -p /mnt/@/var/log
  mkdir -p /mnt/@/var/cache/pacman/pkg
  umount -R /mnt
  ```

- Mount btrfs filesystem
  ```
  mount -o noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@ /dev/nvme0n1p2 /mnt
  mount -o noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@home /dev/nvme0n1p2 /mnt/home
  mount -o noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
  mount -o noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@var_log /dev/nvme0n1p2 /mnt/var/log
  mount -o noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=@pacman_pkgs /dev/nvme0n1p2 /mnt/var/cache/pacman/pkg
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
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /            btrfs   rw,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@                0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /home        btrfs   rw,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@home            0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /.snapshots  btrfs   rw,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@snapshots       0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /var/log     btrfs   rw,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@var_log         0       0
  UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /var/cache/pacman/pkg  btrfs   rw,noatime,compress=zstd:1,space_cache=v2,autodefrag,subvol=/@pacman_pkgs       0       0
  ```

  note that the last tow collums should be `0` not `1`, see [this](https://wiki.archlinux.org/title/Fstab#Usage).
  You can get UUID for `/dev/nvme0n1p1` by running this command `lsblk -dno UUID /dev/nvme0n1p1`, same for `/dev/nvme0n1p2`.

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

