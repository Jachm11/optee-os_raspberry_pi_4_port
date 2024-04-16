# optee-os_raspberry_pi_4_port
This repository contains a way to port the op-tee TEE to rpi4.

# First of all
This repository is based on the findings of this 3 repositories: https://github.com/peter-nebe/optee_os/tree/master, https://github.com/jefg89/optee-rpi4/tree/main and most notably https://github.com/joaopeixoto13/OPTEE-RPI4.
The idea of this repo is to work as a standardized step-by-step guide to include OP-TEE support to the Raspberry Pi 4.

# Needed tools
- Any machine with Linux (that is outside of the RPI4 itself). The steps on this repository where executed on Ubuntu 22.04 LTS.
- Either Yocto or Buildroot. Preferably the later one because in this reporsitory we'll using only Buildroot. You can download it at: https://buildroot.org/
- asda
