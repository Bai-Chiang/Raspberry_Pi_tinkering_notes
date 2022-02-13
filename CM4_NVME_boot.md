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

  Using the `SD Card Copier` copy the current raspberry pi os to nvme ssd, check `New Partition UUIDs`.

