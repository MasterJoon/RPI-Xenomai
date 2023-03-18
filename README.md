# This project is about Xenomai-3.2 and i-Pipe patch to Kernel of Raspberry Pi 3 and 4.

## I referenced following sites.  
[http://www.simplerobot.net/2019/12/xenomai-3-for-raspberry-pi-4.html](http://www.simplerobot.net/2019/12/xenomai-3-for-raspberry-pi-4.html)  
[https://github.com/cpb-/xeno-pi](https://github.com/cpb-/xeno-pi)  

I used WSL (Ubuntu 20.04) in windows 11 PC for Raspberry Pi kernel compiling.
# Host PC
## 1) Install necessary tools
```sh
~$ sudo apt-get install git bc bison flex libssl-dev make
~$ sudo apt-get install build-essential libncurses5-dev
~$ sudo apt-get install g++-arm-linux-gnueabihf
~$ sudo apt-get install gdb-multiarch
```  

## 2) Download Raspberry Pi Kerenl source, Xenomai source and patch files.
```sh
~$ mkdir ./rpi-kernel
~$ cd rpi-kernel/
~/rpi-kernel$ git clone https://github.com/raspberrypi/linux.git
~/rpi-kernel$ git clone https://github.com/MasterJoon/RPI-Xenomai.git
~/rpi-kernel$ wget https://source.denx.de/Xenomai/xenomai/-/archive/v3.2.1/xenomai-v3.2.1.tar.bz2
~/rpi-kernel$ tar xjvf xenomai-v3.2.1.tar.bz2
~/rpi-kernel$ cd linux/
~/rpi-kernel/linux$ git reset --hard c078c64fecb325ee86da705b91ed286c90aae3f6
```  

## 3) Patch the linux kernel source with Xenomai-v3.2.1 and i-Pipe
```sh
~/rpi-kernel/linux$ patch -p1 < ../RPI-Xenomai/pre-rpi4-4.19.86-xenomai3-simplerobot.patch
~/rpi-kernel/linux$ patch -p1 < ../RPI-Xenomai/ipipe-core-4.19.82-arm-6-mod-4.49.86.patch
~/rpi-kernel/linux$ ../xenomai-v3.2.1/scripts/prepare-kernel.sh --arch=arm --linux=./
```  

## 4) Create kernel compile destination directory.
```sh
~/rpi-kernel/linux$ mkdir ../rt-Kernel
~/rpi-kernel/linux$ mkdir ../rt-kernel/boot
```  

## 5) Export variable for kernel compile.
```sh
~/rpi-kernel/linux$ export ARCH=arm
~/rpi-kernel/linux$ export CROSS_COMPILE=arm-linux-gnueabihf-
~/rpi-kernel/linux$ export INSTALL_MOD_PATH=~/rpi-kernel/rt-kernel
~/rpi-kernel/linux$ export INSTALL_DTBS_PATH=~/rpi-kernel/rt-kernel
```  

## 6) Kernel compile configuration and compile.
In the compile process of a general Raspberry Pi kernel, a default configuration file is created with the ```make bcm2711_defconfig``` command using the **arch/arm/configs/bcm2711_defconfig** file (.config), and then the compile settings are additionally modified through the ```make menuconfig``` command.  
However, for the compile of the Xenomai patched kernel, the method of directly modifying the compile setting was not successful. So, I used the method of directly copying the configuration file from the pre-built kernel distributed on the [Simplerobot](https://github.com/thanhtam-h/rpi4-xeno3) site and using it as it is.  
```sh
~/rpi-kernel/linux$ cp ../RPI-Xenomai/config-4.19.86-v7l-ipipe .config
~/rpi-kernel/linux$ make menuconfig
```  
#### In menuconfig, modify following options.  
**<span style="color: #6262F1">G</span>eneral setup --->**  
　↳ **(-v7l-ipipe) <span style="color: #6262F1">L</span>ocal version - append to kernel release**  
　　You can append text in kernel version string. (Optional)  
**<span style="color: #6262F1">K</span>ernel Features --->**  
　↳ **<span style="color: #6262F1">T</span>imer Frequency (100 Hz) --->**  
　　Select "1000 Hz"  
**[*] <span style="color: #6262F1">X</span>enomai/cobalt --->**  
　↳ **<span style="color: #6262F1">C</span>ore features --->**  
　　↳ **(1000) <span style="color: #6262F1">R</span>ound-robin quantum (us) --->**  
　　　Set to '1' (Not sure if it works.)  

```sh
~/rpi-kernel/linux$ make -j4 zImage
~/rpi-kernel/linux$ make -j4 modules
~/rpi-kernel/linux$ make -j4 dtbs
~/rpi-kernel/linux$ make -j4 module_install
~/rpi-kernel/linux$ make -j4 dtbs_install
~/rpi-kernel/linux$ ./scripts/mkknlimg ./arch/arm/boot/zImage$INSTALL_MOD_PATH/boot/kernel7-xeno.img
~/rpi-kernel/linux$ cd ..
~/rpi-kernel$ tar czvf rt-kernel.tgz ./rt-kernel/
```  

## 7) Compile Xenoami Library
```sh
~/rpi-kernel$ cd xenomai-v3.2.1/
~/rpi-kernel/xenomai-v3.2.1$ ./scripts/bootstrap
```
Through the ```./scripts/bootstrap``` command, **configure** file is created. but, in this file, there is a error.  
Comparing Xenoami-3.1 configure file, I found the solution. just remove a line using "FUSE" variable.  
#### Error patch in configure file
```sh
~/rpi-kernel/xenomai-v3.2.1$ nano configure
^W (Ctrl + w)
Search: FUSE,
Add '#' before 'PKG_CHECK_MODULES(FUSE, fuse)'
  ↳ #PKG_CHECK_MODULES(FUSE, fuse)
^O (Ctrl + o)
File Name to Write: configure -> Enter
^X (Ctrl + x)
~/rpi-kernel/xenomai-v3.2.1$ ./configure --host=arm-linux-gnueabihf --enable-smp --with-core=cobalt
~/rpi-kernel/xenomai-v3.2.1$ make
~/rpi-kernel/xenomai-v3.2.1$ sudo make install
~/rpi-kernel/xenomai-v3.2.1$ cd ..
~/rpi-kernel$ tar cjvf xeno3-deploy.tar.bz2 /usr/xenomai
```  

## 8) Transfer the kernel file and Xenomai library
```sh
~/rpi-kernel$ sudo scp rt-kernel.tgz pi@192.168.--.--:~/
~/rpi-kernel$ sudo scp xeno3-deploy.tar.bz2 pi@192.168.--.--:~/
```  

# In Raspberry Pi
## 1) Kernel Change
```sh
~$ tar xzvf rt-kernel.tgz
~$ cd rt-kernel/boot/
~/rt-kernel/boot$ sudo cp -rd * /boot
~/rt-kernel/boot$ cd ../lib/
~/rt-kernel/lib$ sudo cp -dr * /lib
~/rt-kernel/lib$ cd ../overlays/
~/rt-kernel/overlays$ sudo cp -d * /boot/overlays
~/rt-kernel/overlays$ cd ..
~/rt-kernel$ sudo cp -d bcm* /boot
~/rt-kernel$ sync
~/rt-kernel$ cd ..
~$ sudo nano /boot/config.txt
# Append the following line.
kernel=kernel7-xeno.img
~$ sudo nano /boot/cmdline.txt
# Append the following line in a single line.
dwc_otg.fiq_enable=0 dwc_otg.fiq_fsm_enable=0 dwc_otg.nak_holdoff=0 isolcpus=0,1 xenomai.supported_cpus=0x3
```  

## 2) Install Xenoamai library
```sh
~$ sudo tar -xjvf xeno3-deploy.tar.bz2 -C /
~$ sudo nano /etc/ld.so.conf.d/xenomai.conf
# Append the following lines.
# xenomai lib path
/usr/local/lib
/usr/xenomai/lib
~$ sudo ldconfig
~$ sudo reboot
```  

# Xenomai Latency Test
```sh
~$ sudo /usr/xenomai/bin/latency
```  
