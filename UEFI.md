Set up Raspberry Pi UEFI boot, so that we could use universal `aarch64` image to install OS just like PC.

- Download latest raspberry pi 4 UEFI release from [here](https://github.com/pftf/RPi4/releases)
- Extract and edit `config.txt`, for example add this line `dtoverlay=dwc2,dr_mode=host` to enable USB 2.0 output on the CM4 IO board.
- Create an EFI partition on micro SD card on which we will install OS
  ```
  # parted /dev/sdx
  (parted) mklabel gpt
  (parted) mkpart "EFI_partition" fat32 1MiB 513MiB
  (parted) set 1 esp on
  (parted) quit
  # mkfs.fat -F 32 /dev/sdx1
  ```
  where we create a new GPT partition table, and then create a 512MiB EFI partition.
- now we copy the EFI firmware to this EFI partition
  ```
  # mount /dev/sdx1 /mnt
  # cp -r /path/to/RPi4_UEFI_Firmware/. /mnt
  # umount /mnt
  ```
  umplug sd card.
- Download iso (like the [debian](https://www.debian.org/distrib/netinst) arm64 iso) and create USB boot drive as usual
  ```
  dd if=/path/to/image.iso of=/dev/sdx conv=sync status=progress oflag=direct  bs=8M
  ```
- Insert the micro sd card and the USB boot drive to the raspberry pi, it should boot into UEFI, then you can select to boot from usb live iso like PC.

- disable the 3 GB RAM limit if linux kernel version 5.8 or later

  In the UEFI settins: `Device Manager` → `Raspberry Pi Configuration` → `Advanced Settings` and set Limit RAM to 3 GB to <Disabled>
