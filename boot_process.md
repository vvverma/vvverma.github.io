# General Embedded Linux Boot Sequence

* TOC
{:toc}

## Power ON Reset

Apply power Power on reset

## ROM Boot Loader 

Loaded and executed in ROM

After system startup is complete by POST, ROM code will initialize stack pointer and sets up the watchdog, if failed to find SPL it will reboot. After watchdog is enabled,  the PLL (Phase Lock Loop) which is nothing but to step up and generate a wide range of clock frequencies for a distributed subsystem on the SOC is started. Checks the clocking configuration. Then moves forward with Booting checking the external memory or devices list mentioned. ROM copies or loads the SPL into the SRAM from the memory device configured from. ROM reads the header of the image which consists of two important information i.e load addr of the SPL and size of the SPL image. 

Summary for ROM code functions-

* Stack Setup
* Watchdog timer configuration (Sets to 3 minutes timeout)
* System clock configuration using PLL
* Search Memory Devices or other bootable interfaces for boot.bin or MLO or SPL
* Copy MLO/ boot.bin/ SPL into the internal SRAM of the chip
* Execute "MLO" or "SPL"

## Secondary Program Loader Second Stage Bootloader runs out of internal SRAM

Summary of Initialization by SPL -

* Its does UART console initialization to print out the debug message
* Reconfigures the PLL to the desired value
* Initializes the DDR registers to use the DDR memory
* Does muxing configuration of boot peripherals pin, because its next job is to load the u-boot from the boot peripherals 
* Copies u-boot to  the DDR and passes the control to U-boot. U-boot image consist of header which consists of load addr.



## Third Program Loader i.e. UBoot runs out of DDR



--> Initialize some of the peripherals like I2c, NAND, FLASH, ETHERNET, UART, USB, MMC etc, because it supports loading kernel from all these peripherals
--> Load the linux kernel image from various boot sources to the DDR memory of the baord.
--> Boot sources: USB, eMMC, SD card, Ethernet, Serial port, NAND flash etc
--> passing the boot arguments to the kernel. 
--> U-boot looks for uImage (ELF) which is u-boot header plus zimage (ELF) of kernel. mkimage can be used to append u-boot header with kernel
--> header is of 64 bytes
--> also loads the device tree
We can read the linux uImage header on uboot hash shell by loading image in ram and using md memory dump command. imi the header info



## Linux Kernel runs out of DDR



Linux bootstrap loader which uncompresses the Linux kernel.


## Root File System can boot from SD Flash Network RAM e-MMC



Entire boot process can be divided into three parts:-

1.Firmware(Bootlaoder)
2.Kernel
3.user space

Basic boot step are as follows:-

a)Power on
b)Firmware(bootloader start)
c)Kernel Decompression Start
d)User Space Start
e)RC script start
f)Application Start

About Boot Loader:-

A boot loader, sometimes referred to as a boot monitor, is a small piece of software that executes soon after powering up a computer. On your desktop Linux PC you may be familiar with lilo or grub, which resides on the master boot record (MBR) of your hard drive. After the PC BIOS performs various system initializations it executes the boot loader located in the MBR. The boot loader then passes system information to the kernel and then executes the kernel. For instance, the boot loader tells the kernel which hard drive partition to mount as root.

In an embedded system the role of the boot loader is more complicated since these systems do not have a BIOS to perform the initial system configuration. The low level initialization of microprocessors, memory controllers, and other board specific hardware varies from board to board and CPU to CPU. These initializations must be performed before a Linux kernel image can execute.

At a minimum an embedded loader provides the following features:
Initializing the hardware, especially the memory controller.
Providing boot parameters for the Linux kernel.
Starting the Linux kernel
Additionally, most boot loaders also provide “convenience” features that simplify development:
Reading and writing arbitrary memory locations.
Uploading new binary images to the board’s RAM via a serial line or Ethernet
Copying binary images from RAM to FLASH memory

Booting start with start_kernel function in the file init/main.c.
This function performs following steps to initialize the system:-
a)init architecture
b)init interrupts
c)init memory
d)start idle thread
e)rest_init
In rest_init function there is a function knowns as kernel_thread which call call_do _basic_setup fuction.
call_do_basic_setup function calls another function do_initcalls() used to initialize the buses and driver and prepare and mount root file system.
Now the use space program started by spawning “/sbin/init,/etc/init,/bin/init,/bin/sh”.



## Questions Related to Booting

Can we avoid using MLO or SPL by making RBL directly loads u-boot into internal SRAM?
Yes, we can but for that we need to squeeze the size of the u-boot image to <128kb of size. The size of the internal SRAM of SOC is really small. Hence we use MLO or SPL to bring in the U-boot

Can we use RBL to load U-boot directly into DDR Memory of the Board and skip using MLO or SPL?
The reason why ROM boot loader of SOC cannot load Uboot directly to DDR by skipping SPL is as follows-
ROM  code won't be having the idea about what kind of DDR RAM being used in the product to initialize it.  DDR RAM is purely product based. For e.g.
There are three Board/ product manufacturing companies X, Y, Z
X-> designed product with SOC using DDR3 by microchip
Y-> designed product with SOC using DDR2 BY Transcend
Z-> designed product with SOC using no DDR
So, to read/write to/from DDR RAM first the tuning parameters must be set properly and DDR registers must be initialized properly. RBL has no idea in which product which DDR RAM is or will be used. Also, it will not know the tuning parameters of DDR like speed, Bandwidth, clock etc.

Hence, in order to know what DDR RAM is connected, spl code needed to be changed and rebuild the binaries. 

