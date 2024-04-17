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

## Installing buildroot
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

Run the next command to bring the graphical interface for our build:
```
make menuconfig
```
Here we are gonna select the main configuration of our build. Your terminal should look something like this: 

IMAGE IMAGE IMAGE

- Disable getty
```
System Configuration 
(Disable "Run a getty (login prompt) after boot")
```

- Setup a root password. Without this the SSH connection can't be stablished!: 
```
System Configuration ==> Root password
(Type any password string (If using locally, to avoid any keyboard config issues on the RPI try to keep it simple.))
```
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









