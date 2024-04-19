# First of all
This repository is based on the findings of these 3 repositories: https://github.com/peter-nebe/optee_os/tree/master, https://github.com/jefg89/optee-rpi4/tree/main and most notably https://github.com/joaopeixoto13/OPTEE-RPI4.
The idea of this repository is to work as a standardized step-by-step guide to include OP-TEE support to the Raspberry Pi 4 (RPI4). By adding or correcting any missing steps, outdated information and merging good ideas present on those repositories.

# Needed tools
- Any machine with Linux (that is, outside of the RPI4 itself). The steps in this repository where executed on Ubuntu 22.04 LTS. This machine should also have a micro-SD card reader and a ethernet port. (Or a way to use both, such as a type-c docking station).
- A USB-UART capable connection such as a FTDI cable. In this repository a FT232 was used for this purpose.
- (Optional) A RJ45 cable. Also known as a ethernet cable.

# Step -1: DISCLAIMER
***Disclaimer!***

The same applies to the RPI4 as to the RPI3: This port of TF-A and OP-TEE OS is ***NOT SECURE!*** It is provided solely for educational purposes and prototyping.

# Step 0: Read the [joaopeixoto13](https://github.com/joaopeixoto13/OPTEE-RPI4) repository

That repository has a amazingly detailed explanation of everything we'll be doing on the next steps, it is recomended to read the repository before proceeding, so that you understand the things we are doing and isn't just a mindless copy/paste. It is not necessary to follow any of the steps as this repository already does that. 

If you care for non of this and blindly trust strangers on the internet I have also made the linux SD image available [here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/sdcard.img). With this you can jump right to step 6 of this guide.

# Step 1: Generate the rich operating system 

## 1.1 Installing buildroot
Open a console and write:
```
mkdir OPTEE-RPI4
cd OPTEE-RPI4/
```

Make sure you have installed all the requiered packages detailed over on [The Buildroot user manual](https://buildroot.org/downloads/manual/manual.html#requirement).
As of now ```April 2024``` the packages needed can be installed by running:

```
sudo apt update && sudo apt upgrade
sudo apt install sed make binutils build-essential diffutils gcc g++ bash patch gzip bzip2 perl tar cpio unzip rsync file bc findutils
```

Also, if needed, we can install git. Plus some optional packeges such as python and ncurses for the GUI:
```
sudo apt install git
sudo apt install libncurses5 libncurses5-dev
sudo apt install python3
```

We will also need curl later on: 
```
sudo apt install curl
```

Once every dependency has been installed we can clone the buildroot repo:
```
git clone git://git.buildroot.net/buildroot 
cd buildroot/
ls configs/
```

In this step, we should see the configuration files for our board, in this case the ```raspberrypi4_64_defconfig``` file.

Now from the buildroot directory, run the following command:
```
make raspberrypi4_64_defconfig
```
We should see ```configuration written to /.../OPTEE-RPI4/buildroot/.config```.

## 1.2 Configuring the build

First we gotta create a kernel configuration fragment file, ```kernel-optee.cfg```.

```
mkdir linux-rpi
cd linux-rpi/
touch kernel-optee.cfg
```
Then open it and paste this (the file is also available [here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/kernel-optee.cfg)):
```
CONFIG_TEE=y
CONFIG_OPTEE=y
```
Save and close.

Run the next command to bring the graphical interface for our build:
```
cd ../
make menuconfig
```
Here we are gonna select the main configuration of our build. Your terminal should look something like this: 

![Menuconfig](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/images/menuconfig.png)

- Set the Additional configuration fragment files
```
Kernel ==> Additional configuration fragment files --> /.../OPTEE-RPI4/buildroot/linux-rpi/kernel-optee.cfg
```
Make sure to change it to the full path of your working directory.

You can go back to the main menu pressing ```ESC``` twice.

- Disable getty
```
System Configuration ==> Run a getty (login prompt) after boot --> No
```

- Setup a root password. Without this the SSH connection can't be stablished!: 
```
System Configuration ==> Root password -> <Password>
```
**Note:** Change \<Password\> with desired string (If using locally, try to avoid any keyboard config issues on the RPI by keeping it simple).

- Enable **DHCP** and **Dropbear**:
```
Target Packages ==> Networking Applications ==> dhcpcd   --> Yes
Target Packages ==> Networking Applications ==> dropbear --> Yes
```

- Enable **optee-client**:
```
Target Packages ==> Security ==> optee-client --> Yes
```

- In order for the examples to appear we must first setup the bootloader:
```
Bootloaders ==> optee_os --> Yes

and then

Bootloaders ==> optee_os ==> OP-TEE OS version --> Custom Git repository
Bootloaders ==> optee_os ==> URL of custom repository  --> https://github.com/OP-TEE/optee_os.git
Bootloaders ==> optee_os ==> Custom repository version --> 3.20.0
Bootloaders ==> optee_os ==> OP-TEE OS needs host-python-cryptography --> Yes
Bootloaders ==> optee_os ==> Build service TAs and libs  --> No
Bootloaders ==> optee_os ==> Build core --> No
Bootloaders ==> optee_os ==> Target platform (mandatory) --> rpi3
```

- Enable **optee-examples**:
```
Target packages ==> Security ==> optee-examples --> Yes
```

- Enable filesystem compression images:
```
Filesystem images ==> tar the root filesystems --> Yes

and then

Filesystem images ==> tar the root filesystems ==> Compression method --> bzip2
```

Now save the configuration by using `TAB` or the arrow keys. Then exit with double `ESC`.

You should see on your terminal: 
```
*** End of the configuration.
*** Execute 'make' to start the build or try 'make help'.
```

To start the build process, simply run:
```
make -j$(nproc)
```
(This step can take about ~1h to be completed on a laptop with an Intel i7-9750H processor)

Once the process is done, **and if no error occurred**, Buildroot output is stored in a single directory, output/.

Run the following command:
```
ls output/
```
You should see 5 directories ```build/```, ```host/```, ```images/```, ```staging/``` and ```target/```. Each explained on [joaopeixoto13](https://github.com/joaopeixoto13/OPTEE-RPI4) repository.

Of all of these direcotories ```images/``` is the one we trully care for. As it contains the files that will be loaded to our target system the RPI4.

## 1.3 Configuring the kernel

Next, run the following command to configure the kernel:

```
make linux-menuconfig
```
This should bring a similar UI as before.

- Enable **Trusted Execution Enviroment support**:
```
Device Drivers ==> Trusted Execution Environment support --> Yes
```
Then save and exit.

# Step 2: Update the ARM Trusted Firmware
In order for optee to correctly run on the RPI4, support for tge BL32 must be addded. 
To avoid version conflicts a specific commit of the ARM Trusted Firmware is suggested here (most recent commit as of ```April 2024```):

Move to the OPTEE-RPI4 directory and clone the repo:
```
cd ../ 
git clone https://github.com/ARM-software/arm-trusted-firmware

cd arm-trusted-firmware/
git reset --hard 0cf4fda

cd plat/rpi/common
ls
```
You should see several files, one of which should be ```rpi4_bl31_setup.c```. 

Go ahead and open ```rpi4_bl31_setup.c``` and navigate to the ```bl31_early_platform_setup2``` function:

Replace the function with the following code (or copy the whole file on this repo [here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/rpi4_bl31_setup.c)): 
```
void bl31_early_platform_setup2(u_register_t arg0, u_register_t arg1,
				u_register_t arg2, u_register_t arg3)

{
	/*
	 * LOCAL_CONTROL:
	 * Bit 9 clear: Increment by 1 (vs. 2).
	 * Bit 8 clear: Timer source is 19.2MHz crystal (vs. APB).
	 */
	mmio_write_32(RPI4_LOCAL_CONTROL_BASE_ADDRESS, 0);

	/* LOCAL_PRESCALER; divide-by (0x80000000 / register_val) == 1 */
	mmio_write_32(RPI4_LOCAL_CONTROL_PRESCALER, 0x80000000);

	/* Early GPU firmware revisions need a little break here. */
	ldelay(100000);

	/* Initialize the console to provide early debug support. */
	rpi3_console_init();

	/* ------------------------------------------------------ */
	/* NEW CODE FOR OPTEE SUPPORT ADDED HERE */

	/*
	 * [RPI4 CONFIGURATION]
	 * 1) Copy the OP-TEE OS image to the entry address, because the BL32 binaries are stored in the FIP address space (0x2000 => 128k bytes)
	 * 2) Define the Device Tree Blob (DTB) address, calling the function rpi4_get_dtb_address()
	 * 3) Define the Secure State as Secure
	 * 
	 * Note: In this case, the OP-TEE OS image in no longer than 500k bytes
	 */
	const size_t trustedOS_size = 500 * 1024;									// Define the OP-TEE OS image size (500k bytes)
	const void *const fip_addr = (const void*)0x20000;								// Define the OP-TEE OS image load address (FIP address - 0x20000)
	void *const trustedOS_addr = (void*)0x10100000;									// Define the OP-TEE OS image address (Secure Payload - 0x10100000)
	VERBOSE("rpi4: copy trusted_os image (%lu bytes) from %p to %p\n", trustedOS_size, fip_addr, trustedOS_addr);
	memcpy(trustedOS_addr, fip_addr, trustedOS_size);								// Copy the OP-TEE OS image to the entry address

	/* Initialize the OP-TEE OS image info. */
	bl32_image_ep_info.pc = (uintptr_t)trustedOS_addr;								// Define the bl32 entry point address (0x10100000)
	bl32_image_ep_info.args.arg2 = rpi4_get_dtb_address();								// Define the Device Tree Blob (DTB) address
	SET_SECURITY_STATE(bl32_image_ep_info.h.attr, SECURE);								// Define the Secure State
	VERBOSE("rpi4: trusted_os entry: %p\n", (void*)bl32_image_ep_info.pc);
	VERBOSE("rpi4: bl32 dtb: %p\n", (void*)bl32_image_ep_info.args.arg2);

	/* NEW CODE FOR OPTEE SUPPORT ENDS HERE */
	/* ------------------------------------------------------ */

		/* Initialize the Linux kernel image info. */
	bl33_image_ep_info.pc = plat_get_ns_image_entrypoint();
	bl33_image_ep_info.spsr = rpi3_get_spsr_for_bl33_entry();
	SET_SECURITY_STATE(bl33_image_ep_info.h.attr, NON_SECURE);
	VERBOSE("rpi4: kernel entry: %p\n", (void*)bl33_image_ep_info.pc);

#if RPI3_DIRECT_LINUX_BOOT
# if RPI3_BL33_IN_AARCH32
	/*
	 * According to the file ``Documentation/arm/Booting`` of the Linux
	 * kernel tree, Linux expects:
	 * r0 = 0
	 * r1 = machine type number, optional in DT-only platforms (~0 if so)
	 * r2 = Physical address of the device tree blob
	 */
	VERBOSE("rpi4: Preparing to boot 32-bit Linux kernel\n");
	bl33_image_ep_info.args.arg0 = 0U;
	bl33_image_ep_info.args.arg1 = ~0U;
	bl33_image_ep_info.args.arg2 = rpi4_get_dtb_address();
# else
	/*
	 * According to the file ``Documentation/arm64/booting.txt`` of the
	 * Linux kernel tree, Linux expects the physical address of the device
	 * tree blob (DTB) in x0, while x1-x3 are reserved for future use and
	 * must be 0.
	 */
	VERBOSE("rpi4: Preparing to boot 64-bit Linux kernel\n");
	bl33_image_ep_info.args.arg0 = rpi4_get_dtb_address();
	bl33_image_ep_info.args.arg1 = 0ULL;
	bl33_image_ep_info.args.arg2 = 0ULL;
	bl33_image_ep_info.args.arg3 = 0ULL;
# endif /* RPI3_BL33_IN_AARCH32 */
#endif /* RPI3_DIRECT_LINUX_BOOT */
}

```
Save and close the file.

# Step 3: Update OPTEE-OS

First, we need to download the existing OPTEE Trusted OS from the official OPTEE Github website.
Same as before, a specific commit is suggested: 

Move to the OPTEE-RPI4 directory and clone the repo:
```
cd ../../../../
git clone https://github.com/OP-TEE/optee_os

cd optee_os
git reset --hard 6376023
```

Next, execute the following command to create a new platform for the Raspberry Pi 4 based on the previsous version:
```
cd core/arch/arm
cp -rf plat-rpi3 plat-rpi4

cd plat-rpi4
ls
```
Now, you should see four files ```conf.mk```,```main.c```,```platform_config.h``` and ```sub.mk```.

Open the ```platform_config.h``` file (also [here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/platform_config.h)). Two things must be changed in the file: the UART base address and the UART Clock Frequency, like so:

```
#define CONSOLE_UART_BASE	0xfe215040 /* UART0 */
#define CONSOLE_BAUDRATE	115200
#define CONSOLE_UART_CLK_IN_HZ	48000000
```

# Step 4: Compile the ARM Trusted Firmware and the Trusted OS

First we need the right ARM aarch64 toolchains. As detailed in [OPTEE Toolchains](https://optee.readthedocs.io/en/latest/building/toolchains.html).

Go back to the OPTEE-RPI4 directory and download the toolchain:
```
cd ../../../../../
git clone https://github.com/OP-TEE/build.git
cd build/
make -f toolchain.mk -j2
cd ../
```
Now we need to add this new toolchain binaries to our ```PATH```:

Run:
```
export PATH=/.../OPTEE-RPI4/toolchains/aarch64/bin:$PATH
```
Make sure to change it to your working directory path.

Then create and copy [this](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/Makefile) makefile on your OPTEE-RPI4 working directory.
```
touch Makefile
```
Open the make file and copy this. Be sure to change the ```DIR := ``` statement to the path of your working directory, in my case I used Documents:
```
# Define the paths
DIR := /home/user/Documents
OPTEE_DIR := ${DIR}/OPTEE-RPI4
TFA_DIR := ${OPTEE_DIR}/arm-trusted-firmware
RPI4_TFA_DIR := ${TFA_DIR}/plat/rpi/rpi4
OPTEE_OS_DIR := ${OPTEE_DIR}/optee_os

.PHONY: all clean

all: 	
	# Compile the ARM Trusted Firmware
	@echo "Compiling the ARM Trusted Firmware"
	make -C ${TFA_DIR} \
	  CROSS_COMPILE=${DIR}/OPTEE-RPI4/toolchains/aarch64/bin/aarch64-none-linux-gnu- \
	  PLAT=rpi4 \
	  SPD=opteed \
	  DEBUG=1

	# Compile the Trusted OS
	@echo "Compiling the Trusted OS"
	make -C ${OPTEE_OS_DIR} \
	  CROSS_COMPILE=${DIR}/OPTEE-RPI4/toolchains/aarch64/bin/aarch64-none-linux-gnu- \
	  PLATFORM=rpi4 \
	  CFG_ARM64_core=y \
	  CFG_USER_TA_TARGETS=ta_arm64 \
	  CFG_DT=y

	# Change to the ARM Trusted Firmware directory to access the 'bl31.bin'
	mkdir -p ${RPI4_TFA_DIR}/build

	# Copy the binary
	cp ${TFA_DIR}/build/rpi4/debug/bl31.bin ${OPTEE_DIR}/bl31-pad.tmp

	# Truncate the binary to 128k bytes
	truncate --size=128K ${OPTEE_DIR}/bl31-pad.tmp

	# Concatenate the bl31 with the bl32 (or tee-pager_v2)
	cat ${OPTEE_DIR}/bl31-pad.tmp ${OPTEE_OS_DIR}/out/arm-plat-rpi4/core/tee-pager_v2.bin > ${OPTEE_DIR}/bl31-bl32.bin
	rm ${OPTEE_DIR}/bl31-pad.tmp
	@echo "Success"
	
clean:
	# Clean the ARM Trusted Firmware
	@echo "Cleaning the ARM Trusted Firmware"
	make -C ${TFA_DIR} clean
	
	# Clean the Trusted OS
	@echo "Cleaning the Trusted OS"
	make -C ${OPTEE_OS_DIR} clean
	
	@echo "Success"
```
Save and close the file.

You can run ```ls``` and check your dictectory looks like this:
```
/arm-trusted-firmware
/build
/buildroot
Makefile
/optee_os
/toolchains
```

Then simply run: 
```
make
```
This Makefile is responsible to not only to compile the ARM Trusted Firmware and the Trusted OS, but also to concatenate this two binaries into one binary to be loaded in the memory of the target device.

When it finishes correctly you should see this on your terminal:
```
...
...
Success
```
And a new file named ```bl31-bl32.bin```. I have uploaded mine [here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/bl31-bl32.bin).

(This file goes into the RPI4 along side the ```cmd.txt``` and the ```config.txt``` file. We do this later on step 5)

# Step 5: Setup the Raspberry Pi 4

## 5.1 Device Tree fix

For some still unknown reason this configuration fails to load OPTEE into the device tree at boot time. This means we need to manually fix it. 

First we need to create a .dts file. Just like [this](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/optee-fix.dts).

```
touch optee-fix.dts
```
Then, open it and copy this code:
```
/dts-v1/;
/plugin/;

/ {
	fragment@0 {
		target-path = "/";

		__overlay__ {
			firmware {
				optee-fix {
					compatible = "linaro,optee-tz";
					method = "smc";
				};
			};
		};
	};
};
```
Now we gotta compile it using the RPI4's device tree compiler. Make sure to change the commands to the correct directory.
```
cd buildroot/output/build/linux-custom/scripts/dtc
./dtc /.../OPTEE-RPI4/optee-fix.dts > /.../OPTEE-RPI4/optee-fix.dtbo
cd ../../../../../../
ls
```
You should see a new file named ```optee-fix.dtbo```. It is a binary just like [this](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/optee-fix.dtbo).

Now we are going to copy this file over to the firmware overlays on the RPI4. So that it can be run once we define it on the `config.txt` file.
```
cp optee-fix.dtbo buildroot/output/images/rpi-firmware/overlays/
```

## 5.2 Edit config.txt and cmdline.txt

First we are going to create both the ```config.txt``` and ```cmd.txt``` files. Note that this already exist inside the ```images/``` directory on build root, we can also just directly edit them, but for safekeeping we'll copy them over to the directory.

```
touch config.txt
touch cmdline.txt
```
Now open ```config.txt``` and copy these contents, so it looks like [this](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/config.txt):

```
# Enable UART
enable_uart=1
uart_2ndstage=1

# 64-bit
arm_64bit=1

# Loads at 0x00 and executes it in EL3 in AArch64
armstub=bl31-bl32.bin

# Define the Kernel image (Normal World)
kernel=Image

# Implement device tree fix for optee
dtoverlay=optee-fix

# Define the Kernel Address
kernel_address=0x200000

# Device Tree Address
device_tree_address=0x2bd00f00

# Device Tree End (64K bytes)
device_tree_end=0x2bd10f00

# Initial ram file system (end of the device tree) ==> To mount the root filesystem
initramfs rootfs.cpio.gz 0x2bd10f00
```

Now we do the same for cmdline.txt. [Here](https://github.com/Jachm11/optee-os_raspberry_pi_4_port/blob/main/cmdline.txt):
```
root=/dev/mmcblk0p2 rootwait console=tty1 console=ttyS0,115200
```

Now we just copy the files to the ```images/``` directory:
```
cp config.txt buildroot/output/images/rpi-firmware/
cp cmdline.txt buildroot/output/images/rpi-firmware/
```

Here we also copy ```bl31-bl32.bin``` as we just definied it on the ```config.txt``` file:
```
cp bl31-bl32.bin buildroot/output/images/rpi-firmware/
```
***Note:*** Notice that this is the same directory with the ```overlays/``` directory we just copied the .dtbo fix to. We will also be able to see this exact directory on our SD card.

Now that we have done all this changes we need to re-build the linux image for the RPI4.
```
cd buildroot/
make -j$(nproc)
```
Don't worry! This time should only take some minutes to finish.

And thats it, our RPI4 is ready to be flashed!

# Step 6: Flash and test!

## 6.1 Flashing the SD card

Remove the SD card from the RPI4 and insert it into your computer. Make sure to backup any important data as we are about to permanently delete anything and everything inside the SD card. 

Once you are sure there nothing valuable, find your SD card using:
```
cd ../
lsblk
```
In my case my SD card appears as ```/dev/sdb```. Be sure of this information as failing to correctly identify the SD card's name will result on total data loss on the written device.

From the OPTEE-RPI4 directory execute the following command to wipe the data and flash the OS:
```
sudo dd if=buildroot/output/images/sdcard.img of=/dev/sdb
```
After this you should see a message on the terminal similar to: 
```
311297+0 records in
311297+0 records out
159384064 bytes (159 MB, 152 MiB) copied, 21,8533 s, 7,3 MB/s
```
Now you can unmount and remove the SD card. You can re-insert the SD card to check all files where succesfully flashed. There should be 2 partitions. A ```rootfs``` with the typical linux file structure and a boot one, where we should see, among other things:
```
overlays/
--- optee-fix.dtbo
bl31-bl32.bin
cmdline.txt
config.txt
```
Proceed and insert the SD card into the RPI4.

## 6.2 Stablish serial port comunication

For us to see the log messages of OPTEE inside the RPI4 we need to watch the UART serial port, since these messages are not shown on the regular interface of the RPI4.
For this we can use either PuTTY or Picocom. We will be using Picocom but it doesn't make much difference to use one or the other.

This section will require the FTDI cable. Follow the instructions on the image bellow to connect it properly:

![ RPI4 pinout](https://dnycf48t040dh.cloudfront.net/fit-in/840x473/GPIO-diagram-Raspberry-Pi-4.png)
![ Picture with "official" colored wires](https://i.stack.imgur.com/VGTaZ.png)

Once you have plugged the cable to both your PC's usb port and to the pins on the RPI4, run: 

```
sudo apt install picocom
```
In order to get to work, we must add the permission to the dialog user, even using with sudo:
```
sudo usermod -a -G dialout $USER
```
Check if the dialout is on the user group:
```
groups $USER
```
Add the baudrate and the port where the board is connected on our computer:
```
sudo picocom -b 115200 /dev/ttyUSB0
```

If succesfull you should see: 
```
picocom v3.1

port is        : /dev/ttyUSB0
flowcontrol    : none
baudrate is    : 115200
parity is      : none
databits are   : 8
stopbits are   : 1
escape is      : C-a
local echo is  : no
noinit is      : no
noreset is     : no
hangup is      : no
nolock is      : no
send_cmd is    : sz -vv
receive_cmd is : rz -vv -E
imap is        : 
omap is        : 
emap is        : crcrlf,delbs,
logfile is     : none
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready

```
## 6.3 Ready the SSH connection for "headless" use (optional)

If you dont have a extra monitor, or just want the flexibilty SSH allows to have, we'll need to configure a SSH conection with the RPI4. (It's worth it!)

First grab your RJ45 and plug it into both the RPI4 and your computer. And turn on the RPI4. A new ethernet connection should be detected.

In your PC go to (or similar):
```
Settings ==> Network ==> Wired (Ethernet Connectiopn) ==> IPv4
(And disable "automatic (dhcp)" and select "share to other computers")
```
Open a new terminal, ```ctrl+alt+t```.

Then install isc-dhcp-server and start the service:
```
sudo apt install isc-dhcp-server
sudo systemctl restart isc-dhcp-server.service
```
Now go ahead and turn off the RPI4.

You will also need a SSH service running on your PC. 
```
sudo apt-get install openssh-server
sudo systemctl start ssh
```

## 6.4 Turn on the RPI4 (Finally!!)

Now go ahead and turn on the RPI4.

Over on your picocom terminal you should see: 
```
Initialising SDRAM 'Micron' 16Gb x2 total-size: 32 Gbit 3200
Loading recovery.elf hnd: 0x00000000
Failed to read recovery.elf error: 6
Loading start4.elf hnd: 0x00000341
Loading fixup4.dat hnd: 0x00000141
MEM GPU: 76 ARM: 948 TOTAL: 1024
FIXUP src: 128 256 dst: 948 1024
Starting start4.elf @ 0xfec00200

NOTICE:  BL31: v2.10.0(debug):v2.10.0-663-g79da34891-dirty
NOTICE:  BL31: Built : 16:52:44, Apr  1 2024
INFO:    ARM GICv2 driver initialized
INFO:    Changed device tree to advertise PSCI.
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a72: CPU workaround for erratum 859971 was applied
WARNING: BL31: cortex_a72: CPU workaround for erratum 1319367 was missing!
INFO:    BL31: cortex_a72: CPU workaround for CVE 2017_5715 was applied
INFO:    BL31: cortex_a72: CPU workaround for CVE 2018_3639 was applied
INFO:    BL31: cortex_a72: CPU workaround for CVE 2022_23960 was applied
INFO:    BL31: Initializing BL32
I/TC: 
I/TC: No non-secure external DT
I/TC: OP-TEE version: 4.1.0-252-ga471cdecf (gcc version 8.2.1 20180802 (GNU Toolchain for the A-profile Architecture 8.2-2019.01 (arm-rel-8.28))) #2 Wed Apr 17 22:17:06 UTC 2024 aarch64
I/TC: WARNING: This OP-TEE configuration might be insecure!
I/TC: WARNING: Please check https://optee.readthedocs.io/en/latest/architecture/porting_guidelines.html
I/TC: Primary CPU initializing
I/TC: Primary CPU switching to normal world boot
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x200000
INFO:    SPSR = 0x3c9
I/TC: Secondary CPU 1 initializing
I/TC: Secondary CPU 1 switching to normal world boot
I/TC: Secondary CPU 2 initializing
I/TC: Secondary CPU 2 switching to normal world boot
I/TC: Secondary CPU 3 initializing
I/TC: Secondary CPU 3 switching to normal world boot
I/TC: Reserved shared memory is enabled
I/TC: Dynamic shared memory is disabled
I/TC: Normal World virtualization support is disabled
I/TC: Asynchronous notifications are disabled
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 6.1.61-v8 (jachm@Jachm-Linux) (aarch64-buildroot-linux-gnu-gcc.br_real (Buildroot 2024.02-226-g81bb14a935) 12.3.0, GNU ld (GNU Binutils) 2.41) #2 SMP PREEMPT Thu Apr  4 16:15:32 CST 2024
[    0.000000] random: crng init done
[    0.000000] Machine model: Raspberry Pi 4 Model B Rev 1.1
[    0.000000] efi: UEFI not found.
[    0.000000] Reserved memory: created CMA memory pool at 0x000000002c000000, size 64 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000000000000-0x000000003fffffff]
[    0.000000]   DMA32    [mem 0x0000000040000000-0x00000000fbffffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000000000-0x000000000007ffff]
[    0.000000]   node   0: [mem 0x0000000000080000-0x000000003b3fffff]
[    0.000000]   node   0: [mem 0x0000000040000000-0x00000000fbffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000000000-0x00000000fbffffff]
[    0.000000] On node 0, zone DMA32: 19456 pages in unavailable ranges
[    0.000000] On node 0, zone DMA32: 16384 pages in unavailable ranges
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.1 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] psci: SMC Calling Convention v1.4
[    0.000000] percpu: Embedded 29 pages/cpu s79144 r8192 d31448 u118784
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: Spectre-v3a
[    0.000000] CPU features: detected: Spectre-BHB
[    0.000000] CPU features: kernel page table isolation forced ON by KASLR
[    0.000000] CPU features: detected: Kernel page table isolation (KPTI)
[    0.000000] CPU features: detected: ARM erratum 1742098
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] alternatives: applying boot alternatives
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 996912
[    0.000000] Kernel command line: coherent_pool=1M 8250.nr_uarts=1 snd_bcm2835.enable_headphones=0 bcm2708_fb.fbwidth=0 bcm2708_fb.fbheight=0 bcm2708_fb.fbswap=1 smsc95xx.macaddr=DC:A6:32:37:EC:BD vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000  root=/dev/mmcblk0p2 rootwait console=tty1 console=ttyS0,115200
[    0.000000] Dentry cache hash table entries: 524288 (order: 10, 4194304 bytes, linear)
[    0.000000] Inode-cache hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[    0.000000] mem auto-init: stack:all(zero), heap alloc:off, heap free:off
[    0.000000] software IO TLB: area num 4.
[    0.000000] software IO TLB: mapped [mem 0x0000000037400000-0x000000003b400000] (64MB)
[    0.000000] Memory: 3813776K/4050944K available (12544K kernel code, 2174K rwdata, 4080K rodata, 4288K init, 1082K bss, 171632K reserved, 65536K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] ftrace: allocating 40199 entries in 158 pages
[    0.000000] ftrace: allocated 158 pages with 5 groups
[    0.000000] trace event string verifier disabled
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU event tracing is enabled.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=4.
[    0.000000]  Trampoline variant of Tasks RCU enabled.
[    0.000000]  Rude variant of Tasks RCU enabled.
[    0.000000]  Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=4
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: gic_handle_irq
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] arch_timer: cp15 timer(s) running at 54.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0xc743ce346, max_idle_ns: 440795203123 ns
[    0.000000] sched_clock: 56 bits at 54MHz, resolution 18ns, wraps every 4398046511102ns
[    0.000303] Console: colour dummy device 80x25
[    0.000942] printk: console [tty1] enabled
[    0.001011] Calibrating delay loop (skipped), value calculated using timer frequency.. 108.00 BogoMIPS (lpj=216000)
[    0.001053] pid_max: default: 32768 minimum: 301
[    0.001188] LSM: Security Framework initializing
[    0.001395] Mount-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.001463] Mountpoint-cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.002755] cgroup: Disabling memory control group subsystem
[    0.004764] cblist_init_generic: Setting adjustable number of callback queues.
[    0.004799] cblist_init_generic: Setting shift to 2 and lim to 1.
[    0.004977] cblist_init_generic: Setting adjustable number of callback queues.
[    0.005007] cblist_init_generic: Setting shift to 2 and lim to 1.
[    0.005178] cblist_init_generic: Setting adjustable number of callback queues.
[    0.005207] cblist_init_generic: Setting shift to 2 and lim to 1.
[    0.005640] rcu: Hierarchical SRCU implementation.
[    0.005666] rcu:     Max phase no-delay instances is 1000.
[    0.007754] EFI services will not be available.
[    0.008275] smp: Bringing up secondary CPUs ...
[    0.017234] Detected PIPT I-cache on CPU1
[    0.017377] CPU1: Booted secondary processor 0x0000000001 [0x410fd083]
[    0.026383] Detected PIPT I-cache on CPU2
[    0.026502] CPU2: Booted secondary processor 0x0000000002 [0x410fd083]
[    0.035504] Detected PIPT I-cache on CPU3
[    0.035625] CPU3: Booted secondary processor 0x0000000003 [0x410fd083]
[    0.035766] smp: Brought up 1 node, 4 CPUs
[    0.035856] SMP: Total of 4 processors activated.
[    0.035878] CPU features: detected: 32-bit EL0 Support
[    0.035897] CPU features: detected: 32-bit EL1 Support
[    0.035919] CPU features: detected: CRC32 instructions
[    0.036082] CPU: All CPU(s) started at EL2
[    0.036118] alternatives: applying system-wide alternatives
[    0.037839] devtmpfs: initialized
[    0.047637] Enabled cp15_barrier support
[    0.047701] Enabled setend support
[    0.047897] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.047948] futex hash table entries: 1024 (order: 4, 65536 bytes, linear)
[    0.049509] pinctrl core: initialized pinctrl subsystem
[    0.050262] DMI not present or invalid.
[    0.050859] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.053951] DMA: preallocated 1024 KiB GFP_KERNEL pool for atomic allocations
[    0.054257] DMA: preallocated 1024 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.055154] DMA: preallocated 1024 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.055258] audit: initializing netlink subsys (disabled)
[    0.055522] audit: type=2000 audit(0.052:1): state=initialized audit_enabled=0 res=1
[    0.056112] thermal_sys: Registered thermal governor 'step_wise'
[    0.056194] cpuidle: using governor menu
[    0.056421] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.056621] ASID allocator initialised with 32768 entries
[    0.057547] Serial: AMBA PL011 UART driver
[    0.068396] bcm2835-mbox fe00b880.mailbox: mailbox enabled
[    0.084213] raspberrypi-firmware soc:firmware: Attached to firmware from 2023-10-17T15:39:16, variant start
[    0.088228] raspberrypi-firmware soc:firmware: Firmware hash is 30f0c5e4d076da3ab4f341d88e7d505760b93ad7
[    0.101946] KASLR enabled
[    0.131987] bcm2835-dma fe007000.dma: DMA legacy API manager, dmachans=0x1
[    0.136743] iommu: Default domain type: Translated 
[    0.136776] iommu: DMA domain TLB invalidation policy: strict mode 
[    0.137176] SCSI subsystem initialized
[    0.137408] usbcore: registered new interface driver usbfs
[    0.137471] usbcore: registered new interface driver hub
[    0.137544] usbcore: registered new device driver usb
[    0.137904] usb_phy_generic phy: supply vcc not found, using dummy regulator
[    0.138387] pps_core: LinuxPPS API ver. 1 registered
[    0.138414] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.138453] PTP clock support registered
[    0.139460] vgaarb: loaded
[    0.139975] clocksource: Switched to clocksource arch_sys_counter
[    0.140589] VFS: Disk quotas dquot_6.6.0
[    0.140678] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.140857] FS-Cache: Loaded
[    0.141028] CacheFiles: Loaded
[    0.149490] NET: Registered PF_INET protocol family
[    0.150144] IP idents hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.155030] tcp_listen_portaddr_hash hash table entries: 2048 (order: 3, 32768 bytes, linear)
[    0.155122] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.155164] TCP established hash table entries: 32768 (order: 6, 262144 bytes, linear)
[    0.155537] TCP bind hash table entries: 32768 (order: 8, 1048576 bytes, linear)
[    0.156860] TCP: Hash tables configured (established 32768 bind 32768)
[    0.157273] MPTCP token hash table entries: 4096 (order: 4, 98304 bytes, linear)
[    0.157471] UDP hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    0.157631] UDP-Lite hash table entries: 2048 (order: 4, 65536 bytes, linear)
[    0.157978] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.158726] RPC: Registered named UNIX socket transport module.
[    0.158757] RPC: Registered udp transport module.
[    0.158777] RPC: Registered tcp transport module.
[    0.158795] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.158828] PCI: CLS 0 bytes, default 64
[    0.161075] hw perfevents: enabled with armv8_cortex_a72 PMU driver, 7 counters available
[    0.161439] kvm [1]: IPA Size Limit: 44 bits
[    0.162705] kvm [1]: vgic interrupt IRQ9
[    0.162929] kvm [1]: Hyp mode initialized successfully
[    1.181259] Initialise system trusted keyrings
[    1.181711] workingset: timestamp_bits=46 max_order=20 bucket_order=0
[    1.187998] zbud: loaded
[    1.190573] NFS: Registering the id_resolver key type
[    1.190622] Key type id_resolver registered
[    1.190644] Key type id_legacy registered
[    1.190752] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    1.190781] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    1.192048] Key type asymmetric registered
[    1.192081] Asymmetric key parser 'x509' registered
[    1.192167] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 246)
[    1.192433] io scheduler mq-deadline registered
[    1.192463] io scheduler kyber registered
[    1.203145] brcm-pcie fd500000.pcie: host bridge /scb/pcie@7d500000 ranges:
[    1.203202] brcm-pcie fd500000.pcie:   No bus range found for /scb/pcie@7d500000, using [bus 00-ff]
[    1.203296] brcm-pcie fd500000.pcie:      MEM 0x0600000000..0x063fffffff -> 0x00c0000000
[    1.203392] brcm-pcie fd500000.pcie:   IB MEM 0x0000000000..0x00bfffffff -> 0x0400000000
[    1.203984] brcm-pcie fd500000.pcie: setting SCB_ACCESS_EN, READ_UR_MODE, MAX_BURST_SIZE
[    1.204437] brcm-pcie fd500000.pcie: PCI host bridge to bus 0000:00
[    1.204470] pci_bus 0000:00: root bus resource [bus 00-ff]
[    1.204499] pci_bus 0000:00: root bus resource [mem 0x600000000-0x63fffffff] (bus address [0xc0000000-0xffffffff])
[    1.204582] pci 0000:00:00.0: [14e4:2711] type 01 class 0x060400
[    1.204824] pci 0000:00:00.0: PME# supported from D0 D3hot
[    1.208648] pci 0000:00:00.0: bridge configuration invalid ([bus 00-00]), reconfiguring
[    1.208909] pci_bus 0000:01: supply vpcie3v3 not found, using dummy regulator
[    1.209083] pci_bus 0000:01: supply vpcie3v3aux not found, using dummy regulator
[    1.209185] pci_bus 0000:01: supply vpcie12v not found, using dummy regulator
[    1.318090] brcm-pcie fd500000.pcie: link up, 5.0 GT/s PCIe x1 (SSC)
[    1.318192] pci 0000:01:00.0: [1106:3483] type 00 class 0x0c0330
[    1.318272] pci 0000:01:00.0: reg 0x10: [mem 0x00000000-0x00000fff 64bit]
[    1.318538] pci 0000:01:00.0: PME# supported from D0 D3cold
[    1.319010] pci_bus 0000:01: busn_res: [bus 01-ff] end is updated to 01
[    1.319068] pci 0000:00:00.0: BAR 8: assigned [mem 0x600000000-0x6000fffff]
[    1.319103] pci 0000:01:00.0: BAR 0: assigned [mem 0x600000000-0x600000fff 64bit]
[    1.319159] pci 0000:00:00.0: PCI bridge to [bus 01]
[    1.319191] pci 0000:00:00.0:   bridge window [mem 0x600000000-0x6000fffff]
[    1.319597] pcieport 0000:00:00.0: enabling device (0000 -> 0002)
[    1.319820] pcieport 0000:00:00.0: PME: Signaling with IRQ 30
[    1.320303] pcieport 0000:00:00.0: AER: enabled with IRQ 30
[    1.321675] bcm2708_fb soc:fb: Unable to determine number of FBs. Disabling driver.
[    1.321711] bcm2708_fb: probe of soc:fb failed with error -2
[    1.329225] Serial: 8250/16550 driver, 1 ports, IRQ sharing enabled
[    1.332310] iproc-rng200 fe104000.rng: hwrng registered
[    1.332809] vc-mem: phys_addr:0x00000000 mem_base=0x3ec00000 mem_size:0x40000000(1024 MiB)
[    1.344823] brd: module loaded
[    1.352043] loop: module loaded
[    1.352789] Loading iSCSI transport class v2.0-870.
[    1.357894] bcmgenet fd580000.ethernet: GENET 5.0 EPHY: 0x0000
[    1.452120] unimac-mdio unimac-mdio.-19: Broadcom UniMAC MDIO bus
[    1.453130] usbcore: registered new interface driver r8152
[    1.453223] usbcore: registered new interface driver lan78xx
[    1.453295] usbcore: registered new interface driver smsc95xx
[    1.454751] xhci_hcd 0000:01:00.0: enabling device (0000 -> 0002)
[    1.454860] xhci_hcd 0000:01:00.0: xHCI Host Controller
[    1.454903] xhci_hcd 0000:01:00.0: new USB bus registered, assigned bus number 1
[    1.455498] xhci_hcd 0000:01:00.0: hcc params 0x002841eb hci version 0x100 quirks 0x0f00040000000890
[    1.456097] xhci_hcd 0000:01:00.0: xHCI Host Controller
[    1.456134] xhci_hcd 0000:01:00.0: new USB bus registered, assigned bus number 2
[    1.456171] xhci_hcd 0000:01:00.0: Host supports USB 3.0 SuperSpeed
[    1.456525] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.01
[    1.456562] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.456590] usb usb1: Product: xHCI Host Controller
[    1.456614] usb usb1: Manufacturer: Linux 6.1.61-v8 xhci-hcd
[    1.456637] usb usb1: SerialNumber: 0000:01:00.0
[    1.457295] hub 1-0:1.0: USB hub found
[    1.457377] hub 1-0:1.0: 1 port detected
[    1.458261] usb usb2: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.01
[    1.458299] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.458327] usb usb2: Product: xHCI Host Controller
[    1.458350] usb usb2: Manufacturer: Linux 6.1.61-v8 xhci-hcd
[    1.458374] usb usb2: SerialNumber: 0000:01:00.0
[    1.458989] hub 2-0:1.0: USB hub found
[    1.459069] hub 2-0:1.0: 4 ports detected
[    1.460599] dwc_otg: version 3.00a 10-AUG-2012 (platform bus)
[    1.461995] usbcore: registered new interface driver uas
[    1.462104] usbcore: registered new interface driver usb-storage
[    1.462825] mousedev: PS/2 mouse device common for all mice
[    1.468453] sdhci: Secure Digital Host Controller Interface driver
[    1.468488] sdhci: Copyright(c) Pierre Ossman
[    1.469020] sdhci-pltfm: SDHCI platform and OF driver helper
[    1.473789] ledtrig-cpu: registered to indicate activity on CPUs
[    1.474080] SMCCC: SOC_ID: ARCH_SOC_ID not implemented, skipping ....
[    1.474156] hid: raw HID events driver (C) Jiri Kosina
[    1.474386] usbcore: registered new interface driver usbhid
[    1.474413] usbhid: USB HID core driver
[    1.482875] optee: probing for conduit method.
[    1.482947] optee: revision 4.1 (a471cdec)
[    1.499799] optee: initialized driver
[    1.500427] NET: Registered PF_PACKET protocol family
[    1.500542] Key type dns_resolver registered
[    1.501484] registered taskstats version 1
[    1.501572] Loading compiled-in X.509 certificates
[    1.502376] Key type .fscrypt registered
[    1.502404] Key type fscrypt-provisioning registered
[    1.515323] uart-pl011 fe201000.serial: there is not valid maps for state default
[    1.516316] uart-pl011 fe201000.serial: cts_event_workaround enabled
[    1.516476] fe201000.serial: ttyAMA1 at MMIO 0xfe201000 (irq = 35, base_baud = 0) is a PL011 rev2
[    1.516730] serial serial0: tty port ttyAMA1 registered
[    1.524787] bcm2835-aux-uart fe215040.serial: there is not valid maps for state default
[    1.525622] printk: console [ttyS0] disabled
[    1.525744] fe215040.serial: ttyS0 at MMIO 0xfe215040 (irq = 36, base_baud = 62500000) is a 16550
[    1.716008] usb 1-1: new high-speed USB device number 2 using xhci_hcd
[    1.719855] printk: console [ttyS0] enabled
[    1.953115] usb 1-1: New USB device found, idVendor=2109, idProduct=3431, bcdDevice= 4.20
[    1.970054] bcm2835-wdt bcm2835-wdt: Poweroff handler already present!
[    1.974601] usb 1-1: New USB device strings: Mfr=0, Product=1, SerialNumber=0
[    1.981746] bcm2835-wdt bcm2835-wdt: Broadcom BCM2835 watchdog timer
[    1.987898] usb 1-1: Product: USB2.0 Hub
[    1.995640] bcm2835-power bcm2835-power: Broadcom BCM2835 power domains driver
[    2.002973] hub 1-1:1.0: USB hub found
[    2.010213] mmc-bcm2835 fe300000.mmcnr: mmc_debug:0 mmc_debug2:0
[    2.015520] hub 1-1:1.0: 4 ports detected
[    2.019867] mmc-bcm2835 fe300000.mmcnr: DMA channel allocated
[    2.048143] of_cfs_init
[    2.085206] mmc0: SDHCI controller on fe340000.mmc [fe340000.mmc] using ADMA
[    2.086111] of_cfs_init: OK
[    2.142418] mmc1: new high speed SDIO card at address 0001
[    2.344000] usb 1-1.3: new full-speed USB device number 3 using xhci_hcd
[    2.391610] mmc0: new ultra high speed DDR50 SDHC card at address 0001
[    2.501194] usb 1-1.3: New USB device found, idVendor=25a7, idProduct=fa67, bcdDevice= 7.10
[    2.502357] mmcblk0: mmc0:0001 EB1QT 29.8 GiB 
[    2.508983] usb 1-1.3: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[    2.508999] usb 1-1.3: Product: 2.4G Receiver
[    2.516266]  mmcblk0: p1 p2
[    2.517673] usb 1-1.3: Manufacturer: CX
[    2.525207] mmcblk0: mmc0:0001 EB1QT 29.8 GiB
[    2.542845] input: CX 2.4G Receiver as /devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.3/1-1.3:1.0/0003:25A7:FA67.0001/input/input0
[    3.268615] hid-generic 0003:25A7:FA67.0001: input,hidraw0: USB HID v1.10 Keyboard [CX 2.4G Receiver] on usb-0000:01:00.0-1.3/input0
[    3.288793] input: CX 2.4G Receiver Mouse as /devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.3/1-1.3:1.1/0003:25A7:FA67.0002/input/input1
[    3.305117] input: CX 2.4G Receiver as /devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.3/1-1.3:1.1/0003:25A7:FA67.0002/input/input2
[    3.320838] input: CX 2.4G Receiver Consumer Control as /devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.3/1-1.3:1.1/0003:25A7:FA67.0002/input/input3
[    3.396254] input: CX 2.4G Receiver System Control as /devices/platform/scb/fd500000.pcie/pci0000:00/0000:00:00.0/0000:01:00.0/usb1/1-1/1-1.3/1-1.3:1.1/0003:25A7:FA67.0002/input/input4
[    3.413469] hid-generic 0003:25A7:FA67.0002: input,hiddev96,hidraw1: USB HID v1.10 Mouse [CX 2.4G Receiver] on usb-0000:01:00.0-1.3/input1
[    3.430210] EXT4-fs (mmcblk0p2): INFO: recovery required on readonly filesystem
[    3.437739] EXT4-fs (mmcblk0p2): write access will be enabled during recovery
[    3.508325] EXT4-fs (mmcblk0p2): orphan cleanup on readonly fs
[    3.514381] EXT4-fs (mmcblk0p2): recovery complete
[    3.521290] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Quota mode: none.
[    3.530067] VFS: Mounted root (ext4 filesystem) readonly on device 179:2.
[    3.537549] devtmpfs: mounted
[    3.546406] Freeing unused kernel memory: 4288K
[    3.551241] Run /sbin/init as init process
[    3.651840] EXT4-fs (mmcblk0p2): re-mounted. Quota mode: none.
Seeding 256 bits and crediting
Saving 256 bits of creditable seed for next boot
Starting syslogd: OK
Starting klogd: OK
Running sysctl: OK
Starting tee-supplicant: Using device /dev/teepriv0.
OK
Starting network: [    3.884448] bcmgenet fd580000.ethernet: configuring instance for external RGMII (RX delay)
[    3.894068] bcmgenet fd580000.ethernet eth0: Link is Down
udhcpc: started, v1.36.1
udhcpc: broadcasting discover
udhcpc: no lease, forking to background
OK
Starting dhcpcd...
[    7.216872] NET: Registered PF_INET6 protocol family
[    7.223738] Segment Routing with IPv6
[    7.227579] In-situ OAM (IOAM) with IPv6
dhcpcd-10.0.5 starting
DUID 00:01:00:01:c7:92:bc:87:dc:a6:32:37:ec:bd
[    7.270337] 8021q: 802.1Q VLAN Support v1.8
[    7.431728] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[    7.450574] cfg80211: Loaded X.509 cert 'benh@debian.org: 577e021cb980e0e820821ba7b54b4961b8b4fadf'
[    7.460818] cfg80211: Loaded X.509 cert 'romain.perier@gmail.com: 3abbc6ec146e09d1b6016ab9d6cf71dd233f0328'
[    7.471706] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[    7.478507] platform regulatory.0: Direct firmware load for regulatory.db failed with error -2
[    7.487567] cfg80211: failed to load regulatory.db
no interfaces have a carrier
Starting dropbear sshd: OK
[    7.972293] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
[    7.979527] bcmgenet fd580000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/tx
```

The most important lines to look for are: 
```
NOTICE:  BL31: v2.10.0(debug):v2.10.0-663-g79da34891-dirty
NOTICE:  BL31: Built : 16:52:44, Apr  1 2024
INFO:    ARM GICv2 driver initialized
INFO:    Changed device tree to advertise PSCI.
INFO:    BL31: Initializing runtime services
INFO:    BL31: cortex_a72: CPU workaround for erratum 859971 was applied
WARNING: BL31: cortex_a72: CPU workaround for erratum 1319367 was missing!
INFO:    BL31: cortex_a72: CPU workaround for CVE 2017_5715 was applied
INFO:    BL31: cortex_a72: CPU workaround for CVE 2018_3639 was applied
INFO:    BL31: cortex_a72: CPU workaround for CVE 2022_23960 was applied
INFO:    BL31: Initializing BL32
I/TC: 
I/TC: No non-secure external DT
I/TC: OP-TEE version: 4.1.0-252-ga471cdecf (gcc version 8.2.1 20180802 (GNU Toolchain for the A-profile Architecture 8.2-2019.01 (arm-rel-8.28))) #2 Wed Apr 17 22:17:06 UTC 2024 aarch64
I/TC: WARNING: This OP-TEE configuration might be insecure!
I/TC: WARNING: Please check https://optee.readthedocs.io/en/latest/architecture/porting_guidelines.html
I/TC: Primary CPU initializing
I/TC: Primary CPU switching to normal world boot
INFO:    BL31: Preparing for EL3 exit to normal world
INFO:    Entry point address = 0x200000
INFO:    SPSR = 0x3c9
I/TC: Secondary CPU 1 initializing
```

```
Running sysctl: OK
Starting tee-supplicant: Using device /dev/teepriv0.
OK
```
If by any reason this shows:
```
Starting tee-supplicant: Using device /dev/teepriv0.
ERR [141] TSUP:main:870: make_daemon(): -1
```
This (most likely) means the device tree is not being correctly loaded, either ```optee-fix.dtbo``` or a error at ```config.txt```. Other possible culprit is the absence of the optee driver altogether.

Look also for: 
```
Starting dropbear sshd: OK
[    7.972293] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
[    7.979527] bcmgenet fd580000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/tx
```
This means the dropbear SSH service is up and running:

If all these lines seem to be there then congrats, we are up and running. Lets test it!

## 6.5 Connect via SSH (optional)

Open a new terminal ```ctrl+alt+t``` (or the one you used to install open ssh).
If you did step 6.3, upon running the next command you should see the DHCP IP assiged to the RPI4:
```
arp -a
```
In my case it looks like this:
```
? ...
? ...
? (10.42.0.66) at dc:a6:32:37:ec:bd [ether] on enx00e04c681881
? ...
? ...
```
This means the IP is ```10.42.0.66```. Then run:
```
ssh root@10.42.0.66
```

You will see something like this:
```
The authenticity of host '10.42.0.66 (10.42.0.66)' can't be established.
ED25519 key fingerprint is SHA256:Zt9noffr7lDxHmoxv8jt5FOxWsgYrWwSx+eHsYBqTZU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? 

```
Type ```yes``` and enter.

You will be promted to enter the password for root:
```
Warning: Permanently added '10.42.0.66' (ED25519) to the list of known hosts.
root@10.42.0.66's password:
```
Type the password you set up back on step 1.2 and enter.

You should see:
```
#
```
This means you have succesfully logged in!

## 6.6 Test op-tee

---

IGNORE IF USING SSH

If you are not using SSH, upon conecting both a monitor and keyboard to the RPI4 youll be promted to login like this: 
```
Welcome to Buildroot
buildroot login:
```
Just type ```root``` as user. And then the password you defined over on step 1.2.

IGNORE IF USING SSH

---

Now lets test optee, over on the rpi4 or SSH conection type:

```
optee_example_hello_world
```
On the RPI4 you should see:
```
#
# optee_example_hello_world
Invoking TA to increment 42
TA incremented value to 43
# 

```
And on the picocom terminal:
```
D/TA:  TA_CreateEntryPoint:39 has been called
D/TA:  TA_OpenSessionEntryPoint:68 has been called
I/TA: Hello World!
D/TA:  inc_value:105 has been called
I/TA: Got value: 42 from NW
I/TA: Increase value to: 43
I/TA: Goodbye!
D/TA:  TA_DestroyEntryPoint:50 has been called
```
Thats our hello world! (Easy, right?)

Enjoy your RPI4 with OP-TEE. 

**Note:** You can terminate picocom with ```ctrl+a``` and then ```ctrl + x```. As for the SSH session you can terminate it with the command ```exit```.

Once again:
***Disclaimer!***

The same applies to the RPi4 as to the RPi3: This port of TF-A and OP-TEE OS is NOT SECURE! It is provided solely for educational purposes and prototyping.



























