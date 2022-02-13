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
