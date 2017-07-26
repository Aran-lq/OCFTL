## 1. Configure lightnvm on nvme-qume
To get my goal, first step is to know about the detail of nvm and how it works on a device based on qemu. The following is my configuration process:
### Configuration environment
- host: linux 4.10, ubuntu17.04 
- guest: ubuntu 16.04 on nvme-qemu.

### Install nvme-qemu
* Clone nvme-qemu(https://github.com/OpenChannelSSD/qemu-nvme)

`$ git clone https://github.com/OpenChannelSSD/qemu-nvme.git `

* configure the QEMU source and compile, install

```
$ ./configure --enable-linux-aio --target-list=x86_64-softmmu --enable-kvm
$ make -j4
$ make install
```  
### Create nvme virtual machine
#### 1. Create a raw disk
`$ qemu-img create -f raw ubuntu.raw 50G`

A big space since I want to compile kernel in VM.
#### 2. Install a new linux system on the raw disk.
`$ qemu-system-x86_64 -m 2G -enable-kvm ubuntu.raw -cdrom ~/ubuntu.iso `

where -m says memory is 2G, -cdrom find the .iso for install(here is ubuntu16.04). After install, we can boot VM using :

`$ qemu-system-x86_64 -m 4G -enable-kvm ubuntu.raw`
### Compile the linux kernel in Qemu
We need the kernel which support lightnvm and nvme

```
$ git clone https://github.com/OpenChannelSSD/linux.git
$ cd linux
$ git checkout for-next (other branch is ok, make sure if it work)
$ make menuconfig  
```
Make sure that the .config includes:

```
CONFIG_NVM=y
# Expose the /sys/module/lnvm/parameters/configure_debug interface
CONFIG_NVM_DEBUG=y
# Target support (required to expose the open-channel SSD as a block device)
CONFIG_NVM_PBLK=y    
# For NVMe support
CONFIG_BLK_DEV_NVME=y
```
Go on compiling:

```
$ make -j4 > /dev/null 
$ make modules_install 
$ make install
```
After above complete, check **/etc/default/grub**, make sure **GRUB\_HIDDEN\_TIMEOUT** has been commented, if not there isn't a boot menu when reboot. Execute the following and boot a new kernel.

```
$ upgrad-grub
$ shutdown -r now
``` 
### Create a blank document which simulate a nvme device 
`$ dd if=/dev/zero of=blknvme bs=1M count=1024`

