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

## 1.a Installing buildroot
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

## 1.b Configuring the build

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

## 1.c Configuring the kernel

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

```
cd OPTEE-RPI4
git clone https://github.com/ARM-software/arm-trusted-firmware

cd arm-trusted-firmware/
git reset --hard 0cf4fda

cd plat/rpi/common
```

Next open the rpi4_bl31_setup.c and navigate to the bl31_early_platform_setup2 function:
```
code rpi4_bl31_setup.c
```

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
	const size_t trustedOS_size = 500 * 1024;										// Define the OP-TEE OS image size (500k bytes)
	const void *const fip_addr = (const void*)0x20000;								// Define the OP-TEE OS image load address (FIP address - 0x20000)
	void *const trustedOS_addr = (void*)0x10100000;									// Define the OP-TEE OS image address (Secure Payload - 0x10100000)
	VERBOSE("rpi4: copy trusted_os image (%lu bytes) from %p to %p\n", trustedOS_size, fip_addr, trustedOS_addr);
	memcpy(trustedOS_addr, fip_addr, trustedOS_size);								// Copy the OP-TEE OS image to the entry address

	/* Initialize the OP-TEE OS image info. */
	bl32_image_ep_info.pc = (uintptr_t)trustedOS_addr;								// Define the bl32 entry point address (0x10100000)
	bl32_image_ep_info.args.arg2 = rpi4_get_dtb_address();							// Define the Device Tree Blob (DTB) address
	SET_SECURITY_STATE(bl32_image_ep_info.h.attr, SECURE);							// Define the Secure State
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









