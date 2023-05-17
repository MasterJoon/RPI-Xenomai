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
~$ sudo apt-get install autoconf automake libtool
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
### 3-1) USB patch in Raspberry Pi 4b rev. 1.5  
- In the new version of Raspberry Pi 4b (rev. 1.5), the USB 3.0 driver module is not working from kernel 4.19.  
- After April 2021(after kernel ver. 5.4), Raspberry Pi 4 kernel find USB 3.0 driver(VL805) firmware from the EEPROM once at the initial stage of booting. unless the driver firmware is valid, the kernel use the module driver(xhci_hcd) in the kernel source codes.
- After googled about this problem, I found the solution in the [Raspberry Pi kernel github issuses](https://github.com/raspberrypi/linux/issues/3713). from *pelwell*'s reply in 4th, I found the [changed source](https://github.com/raspberrypi/linux/commit/c74b1b53254016fd83b580b8d49bb02d72ce4836) about the loading VL805 firmware from EEPROM.  
### **/drivers/usb/host/pci-quirks.c**  
>add at 21 line
```c
#include <soc/bcm2835/raspberrypi-firmware.h>
```  
>add at 631 line
```c
/* The VL805 firmware may either be loaded from an EEPROM or by the BIOS into
 * memory. If run from memory it must be reloaded after a PCI fundmental reset.
 * The Raspberry Pi firmware acts as the BIOS in this case.
 */
static void usb_vl805_init(struct pci_dev *pdev)
{
#if IS_ENABLED(CONFIG_RASPBERRYPI_FIRMWARE)
	struct rpi_firmware *fw;
	struct {
		u32 dev_addr;
	} packet;
	int ret;

	fw = rpi_firmware_get(NULL);
	if (!fw)
		return;

	packet.dev_addr = (pdev->bus->number << 20) |
		(PCI_SLOT(pdev->devfn) << 15) | (PCI_FUNC(pdev->devfn) << 12);

	dev_dbg(&pdev->dev, "RPI_FIRMWARE_NOTIFY_XHCI_RESET %x", packet.dev_addr);
	ret = rpi_firmware_property(fw, RPI_FIRMWARE_NOTIFY_XHCI_RESET,
			&packet, sizeof(packet));
#endif
}

```  
>add at 1240 line
```c
	if (pdev->vendor == PCI_VENDOR_ID_VIA && pdev->device == 0x3483)
		usb_vl805_init(pdev);
```  
### **include/soc/bcm2835/raspberrypi-firmware.h**
>add at 101 line
```c
	RPI_FIRMWARE_NOTIFY_XHCI_RESET =                      0x00030058,
```  

## 4) Create kernel compile destination directory.
```sh
~/rpi-kernel/linux$ mkdir ../rt-Kernel
~/rpi-kernel/linux$ mkdir ../rt-kernel/boot
~/rpi-kernel/linux$ mkdir ../rt-kernel/overlays
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
However, for the compile of the Xenomai patched kernel, the method of directly modifying the compile setting was not successful. So, I used the method of get the config file from the general kernel in Raspberry Pi OS directly. you can get the config file in the raspberry pi shell.  
```sh
raspberrypi~$ sudo modprobe configs
raspberrypi~$ zcat /proc/config.gz > ./config-rpi4
```  
After transfering the config-rpi4 file to the host PC, copy to the linux folder named to *.config*.
```sh
~/rpi-kernel/linux$ cp ../config-rpi4 ./.config
~/rpi-kernel/linux$ make menuconfig
```  
#### In menuconfig, modify following options.  
**<span style="color: #6262F1">G</span>eneral setup --->**  
　↳ **(-v7l-ipipe) <span style="color: #6262F1">L</span>ocal version - append to kernel release**  
　　You can append text in kernel version string. (Optional)  
**<span style="color: #6262F1">K</span>ernel Features --->**  
　↳ **<span style="color: #6262F1">T</span>imer Frequency (100 Hz) --->**  
　　Select "1000 Hz"  
**<span style="color: #6262F1">C</span>PU Power Management --->**  
　↳ **<span style="color: #6262F1">C</span>PU Frequency scaling --->**  
　　↳ **[ ] <span style="color: #6262F1">C</span>PU Frequency scaling --->**  
　　　Disable  
**[*] <span style="color: #6262F1">X</span>enomai/cobalt --->**  
　↳ **<span style="color: #6262F1">C</span>ore features --->**  
　　↳ **(1000) <span style="color: #6262F1">R</span>ound-robin quantum (us) --->**  
　　　Set to '1' (Not sure if it works.)  
**M<span style="color: #6262F1">e</span>mory Management options --->**  
　↳ **[ ] <span style="color: #6262F1">A</span>llow for memory compaction**  
　　Disable  
　↳ **[ ] <span style="color: #6262F1">C</span>ontiguous Memory Allocator**  
　　Disable  
**<span style="color: #6262F1">K</span>ernel hacking --->**  
　↳ **[ ] <span style="color: #6262F1">K</span>GDB: kernel debugger --->**  
　　Disable  

```sh
~/rpi-kernel/linux$ make -j4 zImage
~/rpi-kernel/linux$ make -j4 modules
~/rpi-kernel/linux$ make -j4 dtbs
~/rpi-kernel/linux$ make -j4 module_install
~/rpi-kernel/linux$ cp arch/arm/boot/dts/*.dtb ../rt-kernel/boot/
~/rpi-kernel/linux$ cp arch/arm/boot/dts/overlays/*.dtb* ../rt-kernel/overlays/
~/rpi-kernel/linux$ cp arch/arm/boot/dts/overlays/README ../rt-kernel/overlays/
~/rpi-kernel/linux$ cp arch/arm/boot/zImage ../rt-kernel/boot/kernel7-xeno.img
~/rpi-kernel/linux$ cd ..
~/rpi-kernel$ tar czvf rt-kernel.tgz ./rt-kernel/
```  

## 7) Compile Xenomai Library
### 7-1) Cross compile
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
### 7-2) Native compile and install in Raspberry Pi
- In the native compile, there is no error about "FUSE" variable
```sh
raspberrypi~$ wget https://source.denx.de/Xenomai/xenomai/-/archive/v3.2.1/xenomai-v3.2.1.tar.bz2
raspberrypi~$ tar xjvf xenomai-v3.2.1.tar.bz2
raspberrypi~$ cd xenomai-v3.2.1
raspberrypi~/xenomai-v3.2.1$ ./scripts/bootstrap
raspberrypi~/xenomai-v3.2.1$ ./configure --enable-smp --with-core=cobalt
raspberrypi~/xenomai-v3.2.1$ make
raspberrypi~/xenomai-v3.2.1$ sudo make install
```

## 8) Transfer the kernel file and Xenomai library
```sh
~/rpi-kernel$ sudo scp rt-kernel.tgz pi@192.168.--.--:~/
~/rpi-kernel$ sudo scp xeno3-deploy.tar.bz2 pi@192.168.--.--:~/
```  

# In Raspberry Pi
## 1) Kernel Change
```sh
raspberrypi~$ tar xzvf rt-kernel.tgz
raspberrypi~$ cd rt-kernel/boot/
raspberrypi~/rt-kernel/boot$ sudo cp * /boot
raspberrypi~/rt-kernel/boot$ cd ../lib/
raspberrypi~/rt-kernel/lib$ sudo cp -dr * /lib
raspberrypi~/rt-kernel/lib$ cd ../overlays/
raspberrypi~/rt-kernel/overlays$ sudo cp * /boot/overlays
raspberrypi~/rt-kernel$ sync
raspberrypi~/rt-kernel$ cd ..
```  
>There is a big issue found on 4G RAM version raspberry pi 4, although LPAE (Large Physical Address Extensions) allows Linux 32 bit can access fully 4G memory, the pcie DMA controller can only access up to 3G RAM. This usually causes problem for USB hub (connected via pcie) especially when user set large GPU memory (GPU always use low memory portion). This become serious on ipipe kernel. Workaround for this issue is to limit usable memory to 3G, add "total_mem=3072" in **/boot/config.txt** file.  
```sh
raspberrypi~$ sudo nano /boot/config.txt
# Append the following line.
kernel=kernel7-xeno.img
total_mem=3072
```  
```sh
raspberrypi~$ sudo nano /boot/cmdline.txt
# Append the following line in a single line.
dwc_otg.fiq_enable=0 dwc_otg.fiq_fsm_enable=0 dwc_otg.nak_holdoff=0 isolcpus=0,1 xenomai.supported_cpus=0x3
```  

## 2) Install Xenoamai library
```sh
raspberrypi~$ sudo tar -xjvf xeno3-deploy.tar.bz2 -C /
raspberrypi~$ sudo nano /etc/ld.so.conf.d/xenomai.conf
# Append the following lines.
# xenomai lib path
/usr/local/lib
/usr/xenomai/lib
raspberrypi~$ sudo ldconfig
raspberrypi~$ sudo reboot
```  

# Xenomai Latency Test
```sh
raspberrypi~$ sudo /usr/xenomai/bin/latency
```  
