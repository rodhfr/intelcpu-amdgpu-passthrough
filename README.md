# AMD GPU PASSTHROUGH IN POP OS 22.04 INTEL CPU
This documentation is about to passthrough a secondary GPU to a QEMU virtual machine running Windows. I will not be covering how to do this with single GPU (albeit is indeed doable you just have to setup software hooks to virt-manager), nor do I will cover CPU pinning for now.

##### My system
* OS: POP-OS 22.04 LTS
* CPU: Intel 9400f
* Primary GPU: RX 580GPU
* Secondary GPU: RX 550
* Motherboard: Asrock Fatal1ty B360M Performance
## Requeriments
* CPU that supports IOMMU/vt-d & vt-x/AMD-Vi (IOMMU is a generic name for Intel VT-d and AMD-Vi.)
* Motherboard that supports IOMMU
* **ENABLE** UEFI settings for IOMMU, VT-d (Intel) or AMD-Vi(AMD). Each motherboard has its own name for IOMMU depending on the CPU [Here is some examples](https://us.informatiweb.net/tutorials/it/bios/enable-iommu-or-vt-d-in-your-bios.html)
* Also check if Vt-x or AMD-V is also enable because this is needed to be able to spin virtual machines anyway
* Your guest GPU ROM must support UEFI (pretty much anything after 2012)


## Recommended
* Basic Linux bash knowledge. Terminal commands, navigating through files, creating and running scripts.
* [Read Arch Wiki article](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) about PCI Passthrough via OVMF
* [Read Gentoo Wiki article](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm#IOMMU_kernel_configuration) about GPU passthrough
* Always have a handy USB stick with some Linux ISOs in it
* Maybe learn a little how to use TTY in case you messes and your system doesn't boot anymore
* Don't do this to your main production machine which you have sensitive data or could not spend a day or two troubleshooting to make this work
* You may want a spare monitor connected to the passthrough GPU, this would be much better experience than SPICE connection that will hurt performance, and vnc is slow. As well as mouse and keyboard that can be passed to virtual machine.
* Don't stick to this guide or any guide at all, your machine certainly have some differences and you probably will need to stray out so watch some youtube guides like these

	* [Single GPU Passthrough Tutorial - KVM/VFIO | BlandManStudios](https://youtu.be/eTWf5D092VY)

	* [GPU Pass-through On Linux/Virt-Manager | Mental Outlaw](https://youtu.be/KVDUs019IB8)

	* [SWITCH TO POPOS 22.04 | How To Set Up A Windows Virtual Machine With Single GPU Passthrough | Frank Laterza](https://youtu.be/MrA9jW6iCuo)

	* (Video In Portuguese, a nice in depth video of a real use case) [Games em Máquina Virtual com GPU Passthrough | Entendendo QEMU, KVM, Libvirt](https://youtu.be/IDnabc3DjYY)



#### Before anything, always update your repositories and system

`sudo apt update && sudo apt upgrade`

#### Install libvirt and dependencies
`sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system virtinst virt-manager ovmf dnsmasq`

#### Download Windows ISO
https://www.microsoft.com/pt-br/software-download/windows10ISO

#### Download the Virtio Windows Drivers 
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

## First, enable IOMMU on your bootloader
Chances are your distro will use another booloader like grub, so search your parameters and be sure to do the next steps corretly for each bootloader. [Gentoo Wiki writes about grub bootloader config file ](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm)

There is two ways of editing the bootloader the oficial way for POP-os with systemd-boot is to update via 

`sudo kernelstub -a "intel_iommu=on iommu=pt video=efifb:off"` 

More about [kernelstub](https://support.system76.com/articles/kernelstub/) `kernelstub --help`

The Manual Way:
1) Pop-OS 22.04 use systemd-boot so the config file will be into /boot/efi/loader/entries folder
2) Login into root user `sudo su -`
3) `cd` into /boot/efi/loader/entries and `ls` to see the config file.
4) `nano /boot/efi/loader/entries/Pop_OS-current.conf`
5) **Append `intel_iommu=on iommu=pt video=efifb:off` to the options line**
efifb:off means the EFI Framebuffer is disabled in the boot. Whereas this is not necessairily needed, it can help, because the EFI framebuffer could try to claim the GPU and then interfere with the GPU passthrough. This also means that there will not be splash screen nor graphical output during the early boot process, as well as graphical flickering or possibly black screens.


This is what mine looks like after the alteration
```bash
title Pop!_OS
linux /EFI/Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b/vmlinuz.efi
initrd /EFI/Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b/initrd.img
options root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt video=efifb:off
```

6) `sudo bootctl update`
To update the settings change.
#### Reboot your system
Reboot is essential so the IOMMU could apply

`sudo reboot`

### Check if IOMMU is working
`sudo dmesg | grep -i -e DMAR -e IOMMU`

Here's my output as an example:
```bash
USER@pop-os:~$ sudo dmesg | grep -i -e DMAR -e IOMMU
[sudo] senha para USER: 
[    0.000000] Command line: initrd=\EFI\Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b\initrd.img root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt video=efifb:off disable_idle_d3=1
[    0.008803] ACPI: DMAR 0x000000007EC57D18 000070 (v01 INTEL  EDK2     00000002      01000013)
[    0.008831] ACPI: Reserving DMAR table memory at [mem 0x7ec57d18-0x7ec57d87]
[    0.030700] Kernel command line: initrd=\EFI\Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b\initrd.img root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt video=efifb:off disable_idle_d3=1
[    0.030806] DMAR: IOMMU enabled
[    0.085244] DMAR: Host address width 39
[    0.085245] DMAR: DRHD base: 0x000000fed91000 flags: 0x1
[    0.085250] DMAR: dmar0: reg_base_addr fed91000 ver 1:0 cap d2008c40660462 ecap f050da
[    0.085253] DMAR: RMRR base: 0x0000007ef0a000 end: 0x0000007f153fff
[    0.085256] DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 0
[    0.085257] DMAR-IR: HPET id 0 under DRHD base 0xfed91000
[    0.085258] DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
[    0.088268] DMAR-IR: Enabled IRQ remapping in x2apic mode
[    0.381951] iommu: Default domain type: Passthrough (set via kernel command line)
[    0.498015] DMAR: No ATSR found
[    0.498016] DMAR: No SATC found
[    0.498018] DMAR: dmar0: Using Queued invalidation
[    0.498054] pci 0000:00:00.0: Adding to iommu group 0
[    0.498067] pci 0000:00:01.0: Adding to iommu group 1
[    0.498078] pci 0000:00:12.0: Adding to iommu group 2
[    0.498090] pci 0000:00:14.0: Adding to iommu group 3
[    0.498097] pci 0000:00:14.2: Adding to iommu group 3
[    0.498107] pci 0000:00:16.0: Adding to iommu group 4
[    0.498117] pci 0000:00:17.0: Adding to iommu group 5
[    0.498133] pci 0000:00:1c.0: Adding to iommu group 6
[    0.498153] pci 0000:00:1f.0: Adding to iommu group 7
[    0.498161] pci 0000:00:1f.3: Adding to iommu group 7
[    0.498169] pci 0000:00:1f.4: Adding to iommu group 7
[    0.498178] pci 0000:00:1f.5: Adding to iommu group 7
[    0.498186] pci 0000:00:1f.6: Adding to iommu group 7
[    0.498191] pci 0000:01:00.0: Adding to iommu group 1
[    0.498195] pci 0000:01:00.1: Adding to iommu group 1
[    0.498216] pci 0000:02:00.0: Adding to iommu group 8
[    0.498233] pci 0000:02:00.1: Adding to iommu group 8
[    0.498255] DMAR: Intel(R) Virtualization Technology for Directed I/O
[    1.808675] AMD-Vi: AMD IOMMUv2 functionality not available on this system - This is not a bug.

```
**If nothing outputs then something is wrong, check the last steps**
### Ensure the IOMMU Groups are Valid
1) Create file `check_iommu.sh` and save with the following code inside:
```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
2) Give permitions and run the script.

`sudo chmod +x check_iommu.sh && ./check_iommu.sh`

If your output is something like this, then you're good.
```bash
USER@pop-os:~/Downloads/sh$ ./check_iommu.sh 
IOMMU Group 0:
	00:00.0 Host bridge [0600]: Intel Corporation 8th Gen Core Processor Host Bridge/DRAM Registers [8086:3ec2] (rev 07)
IOMMU Group 1:
	00:01.0 PCI bridge [0604]: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)
	01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Polaris 20 XL [Radeon RX 580 2048SP] [1002:6fdf] (rev ef)
	01:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Ellesmere HDMI Audio [Radeon RX 470/480 / 570/580/590] [1002:aaf0]
IOMMU Group 2:
	00:12.0 Signal processing controller [1180]: Intel Corporation Cannon Lake PCH Thermal Controller [8086:a379] (rev 10)
IOMMU Group 3:
	00:14.0 USB controller [0c03]: Intel Corporation Cannon Lake PCH USB 3.1 xHCI Host Controller [8086:a36d] (rev 10)
	00:14.2 RAM memory [0500]: Intel Corporation Cannon Lake PCH Shared SRAM [8086:a36f] (rev 10)
IOMMU Group 4:
	00:16.0 Communication controller [0780]: Intel Corporation Cannon Lake PCH HECI Controller [8086:a360] (rev 10)
IOMMU Group 5:
	00:17.0 SATA controller [0106]: Intel Corporation Cannon Lake PCH SATA AHCI Controller [8086:a352] (rev 10)
IOMMU Group 6:
	00:1c.0 PCI bridge [0604]: Intel Corporation Cannon Lake PCH PCI Express Root Port #5 [8086:a33c] (rev f0)
IOMMU Group 7:
	00:1f.0 ISA bridge [0601]: Intel Corporation Device [8086:a308] (rev 10)
	00:1f.3 Audio device [0403]: Intel Corporation Cannon Lake PCH cAVS [8086:a348] (rev 10)
	00:1f.4 SMBus [0c05]: Intel Corporation Cannon Lake PCH SMBus Controller [8086:a323] (rev 10)
	00:1f.5 Serial bus controller [0c80]: Intel Corporation Cannon Lake PCH SPI Controller [8086:a324] (rev 10)
	00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (7) I219-V [8086:15bc] (rev 10)
IOMMU Group 8:
	02:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Lexa PRO [Radeon 540/540X/550/550X / RX 540X/550/550X] **[1002:699f]** (rev c7)
	02:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X] **[1002:aae0]**
```
See that the secondary GPU (RX 550) is isolated in IOMMU group 8, this is important to be able to passthrough the PCI.

There's cases when due to limitations to PCI entries your secondary gpu will still be bound with the CPU but this could be OK. [See Section 2.3.1](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Gotchas) in Arch Wiki 

3) In the specific group from the GPU you want to passthrough there is an ID in the ending of the hardware identification from both video interface and audio interface. Mines were `1002:699f` and `1002:aae0`, yours will differ. Copy this identification somewhere easy, this will be needed in the next step.

### Now we have to create the dummy driver (VFIO)

`sudo nano /etc/modprobe.d/vfio.conf`
```bash
options vfio-pci ids=1002:699f,1002:aae0
softdep vfio-pci pre: vfio-pci
```


Change the id to the ones of your GPU, be sure to type this correctly.
`sudo update-initramfs -u`

Here's my output from the update:
```
USER@pop-os:~$ sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.2.6-76060206-generic
W: Possible missing firmware /lib/firmware/amdgpu/ip_discovery.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/vega10_cap.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/sienna_cichlid_cap.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/navi12_cap.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/psp_13_0_10_ta.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/psp_13_0_10_sos.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/aldebaran_cap.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_imu.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_rlc.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_mec.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_me.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_pfp.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_0_toc.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/sdma_6_0_3.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/sienna_cichlid_mes1.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/sienna_cichlid_mes.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/navi10_mes.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_mes1.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/gc_11_0_3_mes.bin for module amdgpu
W: Possible missing firmware /lib/firmware/amdgpu/smu_13_0_10.bin for module amdgpu
kernelstub.Config    : INFO     Looking for configuration...
kernelstub           : INFO     System information: 

    OS:..................Pop!_OS 22.04
    Root partition:....../dev/sdc3
    Root FS UUID:........aff134fa-9f96-4f4f-9100-5c7635201d9b
    ESP Path:............/boot/efi
    ESP Partition:......./dev/sdc1
    ESP Partition #:.....1
    NVRAM entry #:.......-1
    Boot Variable #:.....0000
    Kernel Boot Options:.quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt vfio-pci.ids=1002:699f,1002:aae0
    Kernel Image Path:.../boot/vmlinuz-6.2.6-76060206-generic
    Initrd Image Path:.../boot/initrd.img-6.2.6-76060206-generic
    Force-overwrite:.....False

b.Installer : INFO     Copying Kernel into ESP
b.Installer : INFO     Copying initrd.img into ESP
b.Installer : INFO     Setting up loader.conf configuration
b.Installer : INFO     Making entry file for Pop!_OS
b.Installer : INFO     Backing up old kernel
b.Installer : INFO     No old kernel found, skipping
```
The missing modules from amdgpu is due to the modern amd graphics cards using the open-source version of the driver instead of the old and deprecated amdgpu proprietary driver.

#### Enable VFIO driver, ensure the driver is loaded early in the boot process.
`sudo nano /etc/initramfs-tools/modules`
Add the following lines to the file
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
#### Load the Kernel Modules

`sudo modprobe vfio`

`sudo modprobe vfio_pci`

`sudo modprobe vfio_iommu_type1`

#### Reboot

`sudo reboot`

### Verifying that the configuration worked
`dmesg | grep -i vfio`

There's output? Great, one step less.
```bash
USER@pop-os:~# dmesg | grep -i vfio
[    0.537257] VFIO - User Level meta-driver version: 0.3
[   69.232340] vfio-pci 0000:02:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
[   81.230056] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x19@0x270
[   81.230078] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x1b@0x2d0
[   81.230093] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x1e@0x370
[ 2847.937076] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x19@0x270
[ 2847.937097] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x1b@0x2d0
[ 2847.937114] vfio-pci 0000:02:00.0: vfio_ecap_init: hiding ecap 0x1e@0x370
```
But may this does not appear in dmesg, it was my case, although can still be usable in the guest virtual machine

You can check with
`lspci -nnk`

In the output search for the secondary GPU:
```bash
02:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Lexa PRO [Radeon 540/540X/550/550X / RX 540X/550/550X] [1002:699f] (rev c7)
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Lexa PRO [Radeon 540/540X/550/550X / RX 540X/550/550X] [1002:0b04]
	Kernel driver in use: vfio-pci
	Kernel modules: amdgpu
02:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X] [1002:aae0]
	Subsystem: Advanced Micro Devices, Inc. [AMD/ATI] Baffin HDMI/DP Audio [Radeon RX 550 640SP / RX 560/560X] [1002:aae0]
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
```
In Kernel driver in use you can see that vfio-pci is assigned, this means the gpu is running off the dummy driver instead of the amdgpu, this is great, means that now you we're off to configure the virtual machine.

### If you run into some virt manager permissions problems to acess the PCI, to Fix VFIO Device Permissions
Upon running my virtual machine i've encountered these errors that are related to device permissions, this what did to resolve.
```
internal error: qemu unexpectedly closed the monitor: 2023-07-22T08:40:55.069190Z qemu-system-x86_64: -device vfio-pci,host=0000:02:00.0,id=hostdev0,bus=pci.5,addr=0x0: vfio 0000:02:00.0: failed to open /dev/vfio/8: Permission denied

Traceback (most recent call last):
  File "/usr/share/virt-manager/virtManager/asyncjob.py", line 72, in cb_wrapper
    callback(asyncjob, *args, **kwargs)
  File "/usr/share/virt-manager/virtManager/asyncjob.py", line 108, in tmpcb
    callback(*args, **kwargs)
  File "/usr/share/virt-manager/virtManager/object/libvirtobject.py", line 57, in newfn
    ret = fn(self, *args, **kwargs)
  File "/usr/share/virt-manager/virtManager/object/domain.py", line 1384, in startup
    self._backend.create()
  File "/usr/lib/python3/dist-packages/libvirt.py", line 1353, in create
    raise libvirtError('virDomainCreate() failed')
libvirt.libvirtError: internal error: qemu unexpectedly closed the monitor: 2023-07-22T08:40:55.069190Z qemu-system-x86_64: -device vfio-pci,host=0000:02:00.0,id=hostdev0,bus=pci.5,addr=0x0: vfio 0000:02:00.0: failed to open /dev/vfio/8: Permission denied
```

1) Add user to kvm group

`sudo usermod -aG kvm $(whoami)`

2) Check the groups you're into

`groups`

3) Set vfio group ownership to kvm
`sudo chown root:kvm /dev/vfio/*`

`sudo chmod 660 /dev/vfio/*`

4) Find the user in which qemu is running (is probably who ran virt-manager first)

`ps aux | grep qemu`

5) Change qemu config to match your user (whoami and groups to know each)
`sudo nano /etc/libvirt/qemu.conf`

```
user = "YOURUSER"
group = "YOURUSER"
```

You have to logout the session to see any change in permitions.

### Activate the default libvirt network
`virsh net-autostart default`

`virsh net-start default`

## Create Virtual Machine
New VM > Local install Media > Browse Local > select the ISO you downloaded earlier > Foward > if there is a prompt about permissions, press yes. > Pick Memory, 4096 MB is bare minimum to windows 10 VM, 8192 should be more ok and half your machine Memory would be ok. The CPU cores doesn't matter will change later. > give it a name and BE SURE to check the box "Customize configuration before install". > Check if hypervisor is KVM, chipset is Q35 and be sure to change firmware to UEFI > CPUs tab check the box "Copy host CPU configuration" and click in Topology. Socket means the number o physical CPU dimms that you have into your motherboard, the cores are the physical cores and threads are the virtual cores, mine i5 9400f has 6 cores so i've put 3 cores and 2 threads (you can always change this things later). > Boot Options tab you set the CDROM to the top of the boot order. > Add Hardware > Storage > Browse for Virtio Driver in bus select VirtIO and apply. Click Begin the Installation > Allow to capture mouse and keyboard (if appears) > Enter the Advanced Install in Windows partitioner and add install the virtio driver, then resume to install Windows normally turning down any telemetry that you possibly can.

When it's done, [download the virtio drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso) and install normally inside windows, You may install virtio-win-guest-tools in addition if gonna use remote desktoping > Turn down the machine.

Change the virtual disk bus type
Edit > Preferences > Enable XML editing
Go to the VM settings and change to XML tab > Edit the XMl to change bus from sata to virtio and adress type from drive to pci and apply
```xml
<disk type="file" device="disk">
  <driver name="qemu" type="qcow2"/>
  <source file="/home/rodhfr/Documents/windows10/win10.qcow2"/>
  <target dev="sda" bus="virtio"/>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</disk>
```

Quick Boot into the Vitual Machine to check if its working

## Add GPU Acelleration to the Virtual Machine (Finally)
Go to Virtual Machine Settings and Add Hardware
In PCI Host Device search for you Secondary GPU (The one you want to passthrough) and add both the VGA than the audio. Some GPU has multiple devices, add them all. To check which IOMMU group are located your GPU, check for the host device numbering like 0000:02:00:0 and 0000:02:00:0, remember that you can always `lspci -nnk` to search for you graphics card.

Boot the virtual machine and test the new graphics card in device manager.
You could now install the proprietary drivers for your GPU inside Windows.




# Revert Changes
### Turn down Virtual Machine in virt manager or with the command
`sudo virsh shutdown [VM_NAME]`

### Remove IOMMU from bootloader
1) `sudo su -`
2) `nano /boot/efi/loader/entries/Pop_OS-current.conf`
3) Exclude the arguments about IOMMU initialization `intel_iommu=on iommu=pt video=efifb:off`
4) `sudo bootctl update` to apply the settings change


Here's my original Pop-os bootloader conf:
```
  GNU nano 6.2                                                             /boot/efi/loader/entries/Pop_OS-current.conf                                                                       
title Pop!_OS
linux /EFI/Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b/vmlinuz.efi
initrd /EFI/Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b/initrd.img
options root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash
```

### information about kernall parameters
https://wiki.archlinux.org/title/Kernel_parameters 
https://wiki.archlinux.org/title/Systemd-boot
```
bootctl update
```

### Remove VFIO driver configuration and update initramfs
`sudo rm /etc/modprobe.d/vfio.conf`
`sudo update-initramfs -u`

### Remove VFIO driver
`nano /etc/initramfs-tools/modules` and comment-out the VFIO lines
```
  GNU nano 6.2              /etc/initramfs-tools/modules *                      
# List of modules that you want to include in your initramfs.
# They will be loaded at boot time in the order below.
#
# Syntax:  module_name [args ...]
#
# You must run update-initramfs(8) to effect this change.
#
# Examples:
#
# raid1
# sd_mod
#vfio
#vfio_iommu_type1
#vfio_pci
#vfio_virqfd
```


--------
# Verify Current Boot Options:
```bash
USER@pop-os:~$ sudo cat /etc/kernelstub/configuration
{
  "default": {
    "kernel_options": [
      "quiet",
      "splash"
    ],
    "esp_path": "/boot/efi",
    "setup_loader": false,
    "manage_mode": false,
    "force_update": false,
    "live_mode": false,
    "config_rev": 3
  },
  "user": {
    "kernel_options": [
      "quiet",
      "loglevel=0",
      "systemd.show_status=false",
      "splash",
      "intel_iommu=on",
      "iommu=pt",
      "vfio-pci.ids=1002:699f,1002:aae0"
    ],
    "esp_path": "/boot/efi",
    "setup_loader": true,
    "manage_mode": true,
    "force_update": false,
    "live_mode": false,
    "config_rev": 3
  }
}
```

```bash
USER@pop-os:~$ sudo cat /proc/cmdline

initrd=\EFI\Pop_OS-aff134fa-9f96-4f4f-9100-5c7635201d9b\initrd.img root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt video=efifb:off disable_idle_d3=1
```

---


Isolation of the guest GPU

In order to isolate the gfx card modify /etc/initramfs-tools/modules via:

sudo nano /etc/initramfs-tools/modules and add:

vfio 
vfio_iommu_type1 
vfio_virqfd 
vfio_pci ids=1002:699f,1002:aae0

modify /etc/modules aswell via: sudo nano /etc/modules and add:

vfio 
vfio_iommu_type1 
vfio_pci ids=1002:699f,1002:aae0




sudo nano /etc/modprobe.d/vfio-pci.conf

options vfio-pci ids=xx:xx.x
options vfio-pci ids=1002:699f,1002:aae0

sudo update-initramfs -u


Since the Windows 10 update 1803 the following additional entry needs to be set (otherwise BSOD) create the kvm.conf file via to add the the following line:

sudo nano /etc/modprobe.d/kvm.conf 

options kvm ignore_msrs=1

sudo update-initramfs -u -k all

after reboot
lspci -nnv




