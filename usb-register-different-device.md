# Ipad Pro as a Display Monitor to Pi400-Keyboard Over USB-C


# Requirements 

* Raspberry Pi 400 Development Board
* Raspbian Pi OS flashed on to SD-Card
* Type C USB cable to connect to ipad and Raspberry Pi 400 (This can be USB-C to USB-A also, if you are connecting USB on Ipad Pro)
* Real VNC App installed on Ipad Pro for GUI / Terminus App installed for Non-GUI
    * Using GUI has a benefit of using PI400 Keyboard and Mouse Support (Here we discuss this)
    * Using Non-GUI, need to use Ipad pro keybaord to type commands

# Step 1: Enable SSH/VNC on Raspberry Pi OS

Click the Raspberry Pi menu icon and choose Preferences > Raspberry Pi Configuration. Click Interfaces and set both SSH and VNC to Enabled. Click OK to close the Raspberry Pi Configuration tool.
You can also use raspi-config command also from command line option

# Step 2: Make sure everything is Up-to-date on Raspiberry Pi OS

```
sudo apt update
sudo apt full-upgrade
sudo reboot

```
# Step 3: Update the /boot/config.txt

Uncommment following lines from /boot/config.txt

```
framebuffer_width=1024
framebuffer_height=768

```
And add following lines to end of /boot/config.txt

```
dtoverlay=dwc2
````

# Step 4: Update the /boot/cmdline.txt

Add a new line below console=serial0, … and add the following:

```
modules-load=dwc2
```

# Step 5: Update the /etc/modules

Add a new line in this file with the following module name -

```
libcomposite
```

# Step 6: Create a static ip for USB0

Let's Prevent Raspberry Pi from choosing its internet address. 
Edit the /etc/dhcpcd.conf file and add the following to end of the file :

```
denyinterfaces usb0
```

**Choose an IP range**

Install dnsmasq:
```
sudo apt install dnsmasq -y
```

Now create a usb file  /etc/dnsmasq.d/usb and place the following script in it - 

```
interface=usb0
dhcp-range=10.55.0.2,10.55.0.6,255.255.255.248,1h
dhcp-option=3
leasefile-ro
```

**Choose an address**

A static IP address to connect to Raspberry Pi from the iPad Pro. Edit /etc/network/interfaces.d/usb0 and add the following script -
````
auto usb0
allow-hotplug usb0
iface usb0 inet static
  address 10.55.0.1
  netmask 255.255.255.248
  ````

**Our Fixed IP address is 10.55.0.1**. We will use this (or raspberrypi.local) to SSH and VNC into Raspberry Pi.


# Script to Run on Boot
Add the script given below to /etc/rc.local before exit 0
P.S: Make sure to create below file with execute permissions chmod +x /root/pi-usb-config.sh


**Filename: pi-usb-config.sh**
```
#bin/bash
# this file is from: https://github.com/ckuethe/usbarmory/wiki/USB-Gadgets
echo "creating composite mass-storage, serial, ethernet, hid..."
#modprobe libcomposite
# assumes a disk image exists here...

#FILE=/home/pi/hardpass/usbdisk.img
#mkdir -p ${FILE/img/d}
#mount -o loop,ro,offset=2048 -t vfat $FILE ${FILE/img/d}

cd /sys/kernel/config/usb_gadget/
mkdir -p multi-device
cd multi-device
#echo '' > UDC
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x0100 > bcdDevice # v1.0.0
echo 0x0200 > bcdUSB # USB2

mkdir -p strings/0x409
echo "fedcba9876543210" > strings/0x409/serialnumber
echo "Vishal's Multi-functional Usb" > strings/0x409/manufacturer
echo "MultiGadget USB" > strings/0x409/product

N="usb0"
mkdir -p functions/acm.$N
mkdir -p functions/ecm.$N
mkdir -p functions/hid.$N
mkdir -p functions/mass_storage.$N

# first byte of address must be even
HOST="48:6f:73:74:50:43" # "HostPC"
SELF="42:61:64:55:53:42" # "BadUSB"
echo $HOST > functions/ecm.$N/host_addr
echo $SELF > functions/ecm.$N/dev_addr

echo 1 > functions/mass_storage.$N/stall
echo 0 > functions/mass_storage.$N/lun.0/cdrom
echo 0 > functions/mass_storage.$N/lun.0/ro
echo 0 > functions/mass_storage.$N/lun.0/nofua
echo $FILE > functions/mass_storage.$N/lun.0/file

echo 1 > functions/hid.usb0/protocol
echo 1 > functions/hid.usb0/subclass
echo 8 > functions/hid.usb0/report_length
echo -ne \\x05\\x01\\x09\\x06\\xa1\\x01\\x05\\x07\\x19\\xe0\\x29\\xe7\\x15\\x00\\x25\\x01\\x75\\x01\\x95\\x08\\x81\\x02\\x95\\x01\\x75\\x08\\x81\\x03\\x95\\x05\\x75\\x01\\x05\\x08\\x19\\x01\\x29\\x05\\x91\\x02\\x95\\x01\\x75\\x03\\x91\\x03\\x95\\x06\\x75\\x08\\x15\\x00\\x25\\x65\\x05\\x07\\x19\\x00\\x29\\x65\\x81\\x00\\xc0 > functions/hid.usb0/report_desc

C=1
mkdir -p configs/c.$C/strings/0x409
echo "Config $C: ECM network" > configs/c.$C/strings/0x409/configuration
echo 250 > configs/c.$C/MaxPower
ln -s functions/acm.$N configs/c.$C/
ln -s functions/ecm.$N configs/c.$C/
ln -s functions/mass_storage.$N configs/c.$C/
ln -s functions/hid.$N configs/c.$C/

# this lists available UDC drivers
ls /sys/class/udc > UDC
```
