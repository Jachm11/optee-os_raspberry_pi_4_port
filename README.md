# First of all
This repository is based on the findings of these 3 repositories: https://github.com/peter-nebe/optee_os/tree/master, https://github.com/jefg89/optee-rpi4/tree/main and most notably https://github.com/joaopeixoto13/OPTEE-RPI4.
The idea of this repo is to work as a standardized step-by-step guide to include OP-TEE support to the Raspberry Pi 4. By pointing out any missing steps, outdated information and merging good ideas present on those repos.

# Needed tools
- Any machine with Linux (that is, outside of the RPI4 itself). The steps in this repository where executed on Ubuntu 22.04 LTS. This machine should also have a micro-SD card reader and a ethernet port. (Or a way to use both, such as a type c docking station).
- A USB-UART capable connection such a FTDI cable. In this repository a FT232 was used for this purpose.
- A RJ45 cable. Also known as a ethernet cable.

# Step 0: Read the [joaopeixoto13](https://github.com/joaopeixoto13/OPTEE-RPI4) repo

That repo has amazingly detailed explanation of everything we'll be doing on the next steps, it is recomended to read the repo before proceeding. It is not necesarry to follow any of the steps as this repo already does that. 

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

Also we can install git plus some optional packeges such as python and ncurses for the GUI:
```
sudo apt install git
sudo apt install libncurses5 libncurses5-dev
sudo apt install python3
```

Once every dependency has been installed we can clone the buildroot repo:
```
git clone git://git.buildroot.net/buildroot 
cd buildroot/
ls
ls configs/
```

In this step, we should see the configuration files for our board, in this case the raspberrypi4_64_defconfig file.

Now from the buildroot directory, run the following command:
```
make raspberrypi4_64_defconfig
```
We should see ```configuration written to .../OPTEE-RPI4/buildroot/.config```.

## 1.2 Configuring the build

Run the next command to bring the graphical interface for our build:
```
make menuconfig
```
Here we are gonna select the main configuration of our build. Your terminal should look something like this: 

IMAGE IMAGE IMAGE

- Disable getty
```
System Configuration ==> Run a getty (login prompt) after boot --> No
```

- Setup a root password. Without this the SSH connection can't be stablished!: 
```
System Configuration ==> Root password -> <Password>
```
**Note:** Change <Password> with desired string (If using locally, try to avoid any keyboard config issues on the RPI by keeping it simple).

You can go back to the main menu pressing ```ESC``` twice.

- Enable **DHCP** and **Dropbear**:
```
Target Packages ==> Networking Applications ==> dhcpcd   --> Yes
Target Packages ==> Networking Applications ==> dropbear --> Yes
```

- Enable **optee-client**:
```
Target Packages ==> Security ==> optee-client --> Yes
```
- Enable **optee-examples**:
```
Target packages ==>  Security ==> optee-examples --> Yes
```

- In order for the examples to work we must setup the bootloader as well: TEST TEST TEST
```
Bootloaders ==> optee_os --> Yes

and then

Bootloaders ==> optee_os ==> OP-TEE OS version --> Custom Git repository
Bootloaders ==> optee_os ==> URL of custom repository  --> https://github.com/OP-TEE/optee_os.git
Bootloaders ==> optee_os ==> Custom repository version --> 3.20.0
Bootloaders ==> optee_os ==> OP-TEE OS needs host-python-cryptography --> Yes
Bootloaders ==> optee_os ==> Build core --> No
Bootloaders ==> optee_os ==> Build service TAs and libs  --> No
Bootloaders ==> optee_os ==> Target platform (mandatory) --> rpi3
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
You should see 5 directories ```images/```,```build/```,```host/```,```staging/```,```target/```. All explained on [joaopeixoto13](https://github.com/joaopeixoto13/OPTEE-RPI4) repo.

Of all of these ```images/``` is the one we trully care for. It contains the files that will be loaded to our target system the RPI4.

## 1.3 Configuring the kernel

Next, run the next command to configure the kernel:

```
make linux-menuconfig
```

- Enable **Trusted Execution Enviroment support**:
```
Device Drivers ==> Trusted Execution Environment support --> Yes
```

# Step 2: Update the ARM Trusted Firmware
In order for optee to correctly run on the RPI4 support for tge BL32 must be addded. 
To avoid version conflicts a specific commit of the ARM Trusted Firmware is suggested here (most recent commit as of ```April 2024```):

Move to the OPTEE-RPI4 directory and clone the repo:
```
cd ../ 
git clone https://github.com/ARM-software/arm-trusted-firmware

cd arm-trusted-firmware/
git reset --hard 0cf4fda

cd plat/rpi/common
```

Next open the ```rpi4_bl31_setup.c``` and navigate to the bl31_early_platform_setup2 function:

Replace the function with the following code (or copy the whole file on this repo [here](link)): 
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

```

# Step 3: Update the ARM Trusted Firmware

First, we need to download the existing OPTEE Trusted OS from the official OPTEE Github website:
Same as before a specific commit is suggested: 

Move to the OPTEE-RPI4 directory and clone the repo:
```
cd ../../../../
git clone https://github.com/OP-TEE/optee_os

cd optee_os
git reset --hard 6376023
```

Next, execute the following command to create a new platform for the Raspberry Pi 4 based on the previsous version:
```
cd optee_os/core/arch/arm
cp -rf plat-rpi3 plat-rpi4

cd plat-rpi4
ls
```
Now, you should see four files ```conf.mk```,```main.c```,```platform_config.h``` and ```sub.mk```.

Open the ```platform_config.h``` file (also [here](link)). Two things must be changed in the file. These things are:

```
The UART base address: 0xfe215040
The UART Clock Frequency: 48000000
```

# Step 4: Compile the ARM Trusted Firmware and the Trusted OS

First we need the right ARM aarch64 toolchains. Check [Toolchains](https://optee.readthedocs.io/en/latest/building/toolchains.html)

Go back to the OPTEE-RPI4 directory and download the toolchain:
```
cd ../../../../../
mkdir toolchains
cd toolchains/
wget https://developer.arm.com/-/media/Files/downloads/gnu-a/8.2-2019.01/gcc-arm-8.2-2019.01-x86_64-aarch64-linux-gnu.tar.xz
mkdir aarch64
tar xf gcc-arm-8.2-2019.01-x86_64-aarch64-linux-gnu.tar.xz -C aarch64 --strip-components=1
```
Now we need to add this new binaries to our ```PATH```:

Run:
```
export PATH=/.../OPTEE-RPI4/toolchains/aarch64/bin:$PATH
```
Make sure to change it to your working directory path.

Then create and copy [this](link) makefile on your OPTEE-RPI4 working directory.
```
cd ../
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

You can run ```ls``` and check your dictectory looks like this:
```
arm-trusted-firmware
buildroot
Makefile
optee_os
toolchains
```

Then simply run: 
```
make
```
This Makefile is responsible to not only to compile the ARM Trusted Firmware and the Trusted OS, but also to concatenate this two binaries into one binary to be loaded in the memory of the target device.

When it finishes correctly you should see this on your terminal:
```
Success
```
And a new file named ```bl31-bl32.bin```. I have uploaded mine [here](link).

(This file goes into the RPI4 along side the ```cmd.txt``` and the ```config.txt``` file. We do this later on step 5)

# Step 5: Setup the Raspberry Pi 4

## 5.1 Device Tree fix

For some still unknown reason this configuration fails to load OPTEE into the device tree at boot time. This means we need to manually fix it. 

First we need to create a .dts file. Just like [this](link).

```
touch optee-fix.dts
```
After copy this code:
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
cd buildroot/output/build/linux-custom/scripts/dts
./dtc .../OPTEE-RPI4/optee-fix.dts > .../OPTEE-RPI4/optee-fix.dtbo
cd ../../../../../../
ls
```
You should see a new file named ```optee-fix.dtbo```. 

Now we are going to copy this file over to the firmware overlays on the RPI4. So that it can be run once we define it on the `config.txt` file.
```
cp optee-fix.dtbo buildroot/output/images/rpi-firmware/overlays/
```

## 5.2 Edit config.txt and cmdline.txt

First we are going to create both the config.txt and cmd.txt files. Note that this already exist inside the ```images/``` directory on build root, we can also just directly edit them, but for safekeeping we'll copy them over to the directory.

```
touch config.txt
touch cmdline.txt
```
Now open ```config.txt``` and copy these contents, so it looks like [this](link):

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

Now we do the same for cmdline.txt. [Here](link):
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
***Note:***: Notice that this is the same directory with the ```overlays/``` directory we copied the .dtbo fix to.

Now that we have done all this changes we need to re-build the image.
```
cd buildroot/
make -j$(nproc)
```
Don't worry! This time should only take some seconds to finish.

And thats it, our RPI4 is ready to be flashed!

# Step 6: Flash and test!

## 6.1 Flashing the SD card

Remove the SD card and insert it into the computer. Make sure to backup any important data as we are about to permanently delete anything and everything inside the SD card. 

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

























