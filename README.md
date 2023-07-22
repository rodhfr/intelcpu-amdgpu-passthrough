Requeriments
* CPU that supports IOMMU/vt-d & vt-x/AMD-Vi (IOMMU is a generic name for Intel VT-d and AMD-Vi.)
* Motherboard that supports IOMMU (each CPU manufacturer has its own name, so in the UEFI settings of the motherboard the name is one of the three.)
* GPU ROM (dump you own or download from https://www.techpowerup.com/vgabios/) Bewaring when dumping gpu ROM, the process is dangerous, search about this...

sudo apt update
sudo apt upgrade

## Install libvirt and dependencies

sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system virtinst virt-manager ovmf


### first we need to enable IOMMU on your bootloader
Probably your distro use another booloader so https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
In Pop Os 22.04 we use systemd-boot so the config file will be into /boot/efi/loader/entries folder
append *intel_iommu=on iommu=pt video=efifb:off* to the options line


sudo su -
cd /boot/efi/loader/entries
ls and nano or vim into the config
`options root=UUID=aff134fa-9f96-4f4f-9100-5c7635201d9b ro quiet loglevel=0 systemd.show_status=false splash intel_iommu=on iommu=pt video=efifb:off`
sudo reboot (is important to reboot, so the IOMMU could apply)

### Check if IOMMU is working
sudo dmesg | grep -i -e DMAR -e IOMMU
*my output*
```bash
rodhfr@pop-os:~$ sudo dmesg | grep -i -e DMAR -e IOMMU
[sudo] senha para rodhfr: 
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




### now we have to create the dummy driver (vfio)


sudo nano /etc/modprobe.d/vfio.conf
```
options vfio-pci ids=1002:699f,1002:aae0
softdep vfio-pci pre: vfio-pci
```




