# First of all
This repository is based on the findings of these 3 repositories: https://github.com/peter-nebe/optee_os/tree/master, https://github.com/jefg89/optee-rpi4/tree/main and most notably https://github.com/joaopeixoto13/OPTEE-RPI4.
The idea of this repo is to work as a standardized step-by-step guide to include OP-TEE support to the Raspberry Pi 4. By pointing out any missing steps, outdated information and merging good ideas present on those repos.

# Needed tools
- Any machine with Linux (that is, outside of the RPI4 itself). The steps in this repository where executed on Ubuntu 22.04 LTS.
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
Here we are gonna selected the main configuration of our build.


