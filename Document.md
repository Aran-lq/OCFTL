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
$ ./configure --python=/usr/bin/python2 --enable-kvm --target-list=x86_64-softmmu --enable-linux-aio --enable-debug(optional for gdb) --prefix=(dir of qemu) 
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

Back to host machine, create a blank document using:
 
`$ dd if=/dev/zero of=blknvme bs=1M count=1024`

Using qemu to boot a VM, following is my boot shell script, make sure all the things related are in the same directory.

```
#!/bin/bash
# Boot vm

sudo qemu-system-x86_64 -m 4G -smp 1 --enable-kvm \
-drive file=blknvme,if=none,id=mynvme \
-redir tcp:37812::22 \ (for ssh connection between host and guest, port 37812)
-device nvme,drive=mynvme,serial=deadbeef,namespaces=1,lver=1,lmetasize=16,ll2pmode=0,nlbaf=5,lba_index=3,mdts=10,lnum_lun=16,lnum_pln=2,lsec_size=4096,lsecs_per_pg=4,lpgs_per_blk=512,lbbtable=bbtable.qemu,lmetadata=meta.qemu,ldebug=1 \
ubuntu.raw
```
### Install nvme-cli on guest to feel Lightnvm work

Now, we have 
- a kernel which support *lightnvm* and support *nvme* device.
- a nvme device(simulated by qemu)

so, we need a tools to manage and control our device, **nvme-cli** is suitable for us. Make sure you get a newest version(1.3.44 now) which support *nvme lnvm* command

```
$ git clone https://github.com/OpenChannelSSD/qemu-nvme.git
$ cd qemu-nvme
$ make -j2
$ make install
```
So far we have finished. Now we can use the following command to detect our device and create a target on it.
the following is my output:

```
$ nvme lnvm list
Number of devices: 1
Device      	Block manager	Version
nvme0n1     	gennvm      	(1,0,0)

$ nvme lnvm info
LightNVM (1,0,0). 1 target type(s) registered.
Type	Version
pblk	(1,0,0)

$ lsblk (list available blk devick)
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0      11:0    1 1024M  0 rom  
fd0       2:0    1    4K  0 disk 
sda       8:0    0   50G  0 disk 
├─sda2    8:2    0    1K  0 part 
├─sda5    8:5    0    2G  0 part [SWAP]
└─sda1    8:1    0   48G  0 part /
nvme0n1 259:0    0 1022M  0 disk (our nvme device)

$ nvme lnvm create -d nvme0n1 -n mydevice -t pblk -b 0 -e 3(create a target on nvme device,name mydevice,lun0-lun4)

$ lsblk 
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sr0       11:0    1 1024M  0 rom  
fd0        2:0    1    4K  0 disk 
mydevice 259:1    0  144M  0 disk (**target created by us**)
sda        8:0    0   50G  0 disk 
├─sda2     8:2    0    1K  0 part 
├─sda5     8:5    0    2G  0 part [SWAP]
└─sda1     8:1    0   48G  0 part /
nvme0n1  259:0    0 1022M  0 disk 

```

### References
[Official docs](http://openchannelssd.readthedocs.io/en/latest/gettingstarted/)

http://www.jianshu.com/p/8e11fa93411a

https://hyunyoung2.github.io/2016/10/04/LightNVM_With_OpenChannelSSD_On_QEMU/

















