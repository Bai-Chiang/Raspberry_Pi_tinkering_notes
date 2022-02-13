- [update boot order](https://www.raspberrypi.com/documentation/computers/compute-module.html#cm4bootloader) on ubuntu PC
  ```
  sudo apt install git libusb-1.0-0-dev build-essential
  git clone --depth=1 https://github.com/raspberrypi/usbboot
  cd usbboot
  make
  ```
  
  Now we modify CM4 bootloader
  ```
  cd recovery
  ```
  edit `boot.conf`, set
  ```
  BOOT_ORDER=0xf561
  ```
  This let CM4 first try SD card, then NVME,then USB 2.0. See [this](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-4-bootloader-configuration) for more options.
  
  then run
  ```
  ./update-pieeprom.sh
  sudo ../rpiboot -d . 
  ```
  Now connect raspberry pi CM4 to PC (make sure not boot CM4).
  
  Finally, flash bootloader (you may need to replug the CM4, if the script shows Waiting for BCM2835/6/7/2711...)
  ```
  cd ..
  sudo ./rpiboot -d recovery
  ```
