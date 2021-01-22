[Back to Main Page](README.md)
# Ipad Pro as a Display to Pi400-Keyboard Over USB-C

* TOC
{:toc}

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
#!/bin/bash
cd /sys/kernel/config/usb_gadget/
mkdir -p pi400
cd pi400
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction Composite Gadget
echo 0x0100 > bcdDevice # v1.0.0
echo 0x0200 > bcdUSB # USB2
echo 0xEF > bDeviceClass
echo 0x02 > bDeviceSubClass
echo 0x01 > bDeviceProtocol
mkdir -p strings/0x409
echo "fedcba9876543211" > strings/0x409/serialnumber
echo "Vishal Verma" > strings/0x409/manufacturer
echo "PI400 USB Device" > strings/0x409/product
mkdir -p configs/c.1/strings/0x409
echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower

# Add functions here
# see gadget configurations below
# End functions

mkdir -p functions/ecm.usb0
HOST="00:dc:c8:f7:75:14" # "HostPC"
SELF="00:dd:dc:eb:6d:a1" # "BadUSB"
echo $HOST > functions/ecm.usb0/host_addr
echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/ecm.usb0 configs/c.1/
udevadm settle -t 5 || :
ls /sys/class/udc > UDC
ifup usb0
service dnsmasq restart
```

# References 
* [Ben Hardill's Website](https://www.hardill.me.uk/wordpress/2019/11/02/pi4-usb-c-gadget/)


