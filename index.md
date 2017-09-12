# GPU Passthrough

## Brief
This guide is a simple walk through on how to passthrough a Nvidia GPU to a Windows guest running on a Ubuntu host. This guide should also work for AMD cards, since they have been traditionally easier to passthrough.

## Hardware and OS
* Intel i7-5820K
* Nvidia GTX 1070 ZOTAC
* X99 Chipset Mobo MSI X99S SLI Plus LGA 2011-v3
* Primary GPU MSI Radeon HD 5450
* Ubuntu 16.04 Kernel 4.13

Make sure that the GPU you want to passthrough is not the primary GPU. In most cases this requires putting the GPU not in Slot 1 while in other cases you can edit this in the BIOS.

## Steps

### Bios
1. Enable Virtualization support such as Intel Vt-D, AMD-Vi, or any IOMMU setting.
2. If possible disable any halt on VGA errors. This may allow you to passthrough your primary GPU.

### Modules
You should add the following entries into this file `/etc/initram-fs/modules`
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
Also do not forget to run this command ```sudo update-initramfs -u``` anytime that you modify a file in this folder. 

Next run the following command to list all of the devices on your system and to grep for nvidia devices (change to radeon for AMD cards) ```lspci -nnk | grep -i nvidia``` 
You should see output similar to this

**Nvidia**
```
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:1b81] (rev a1)
	Kernel modules: nvidiafb, nouveau
03:00.1 Audio device [0403]: NVIDIA Corporation Device [10de:10f0] (rev a1)
```

**Radeon**
```
05:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Cedar [Radeon HD 5000/6000/7350/8350 Series] [1002:68f9]
	Subsystem: Micro-Star International Co., Ltd. [MSI] Cedar [Radeon HD 5000/6000/7350/8350 Series] [1462:2127]
	Kernel driver in use: radeon
	Kernel modules: radeon
05:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Cedar HDMI Audio [Radeon HD 5400/6300 Series] [1002:aa68]
	Subsystem: Micro-Star International Co., Ltd. [MSI] Cedar HDMI Audio [Radeon HD 5400/6300 Series] [1462:aa68]
```
From this output you should grab the unique IDs of both the GPU and the audio device to isolate them from the host. In the above example the Nvidia IDs are **10de:1b81** and **10de:10f0**. In the Radeon example they are **1002:68f9** and **1002:aa68**.

Next you will want to edit the following file ```/etc/modprobe.d/local.conf``` and add the following snippet with your IDs in place
```
options vfio-pci ids=10de:1b81,10de:10f0
options vfio-pci disable_vga=1
```
### GRUB
These are the settings that I used for an Intel cpu with Nvidia GPU. If you use a Radeon you should blacklist the radeon driver as well.
```
GRUB_CMDLINE_LINUX_DEFAULT="modprobe.blacklist=nouveau modprobe.blacklist=nvidiafb"
GRUB_CMDLINE_LINUX="intel_iommu=on iommu=pt rd.driver.pre=vfio-pci video=efifb:off vfio_iommu_type1.allow_unsafe_interrupts=1"
```

After  adding these entries do not forget to run `sudo update-grub` to apply the changes to the next reboot.

### KVM + QEMU
I would recommend adding a PPA to have the latest version of qemu and libvirt. 
```
sudo apt-get install qemu-kvm libvirt-bin bridge-utils virtinst ovmf qemu-utils
```
The versions this guide is using:
* libvirtd (libvirt) 2.2.0
* virsh 2.2.0
* qemu-system-x86_64 2.6.2

### Final Steps
Reboot your machine once you have applied all of the above steps. Once you have rebooted your machine you need to verify that the GPU was isolated from your host and is ready for passthrough. Run this command once again `lspci -nnk | grep -i nvidia` and check that the output contains the following line under each device.
```
Kernel driver in use: vfio-pci
```
Such as in this example:
```
03:00.0 VGA compatible controller [0300]: NVIDIA Corporation Device [10de:1b81] (rev a1)
	Subsystem: ZOTAC International (MCO) Ltd. Device [19da:1435]
	Kernel driver in use: vfio-pci
	Kernel modules: nvidiafb, nouveau
```
At this point you can create your VM using virt-manager. You should also boot it and install the OS such as Windows using the KVM graphics card at first. Once you have installed the OS you can passthrough the device under `Add Hardware` and `PCI Host Device` make sure that you pass both the GPU and the audio device through.

Next you need to edit the VM XML to add a few settings to mask the virtualized environment from the GPU (only if using a Nvidia device). Run `virsh edit NAME_OF_VM` and add the following snippet under the `<features>` tag.
```
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='4096'/>
      <vendor_id state='on' value='somestring'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
```
At this point you can go ahead and remove the virtual vga device from the VM and plug a monitor in the GPU that you passed through. Once you click the start button you should see the BIOS appear on your external monitor and then hopefully the OS boots.

### OS does not boot
If you happen to run into a black screen right after the BIOS and or OS logo and there is CPU activity in the VM monitor. Then you can dump the GPU rom and add it to the VM XML file as a separate value. The above Nvidia example had an ID of **03:00.0** and to dump the ROM you would navigate to `/sys/bus/pci/devices/0000\:03\:00.0` and you would run these commands.
```
echo "0000:03:00.0" | tee /sys/bus/pci/drivers/vfio-pci/unbind
```

The above command may not be need but it doesn't hurt to run it to confirm that the GPU is unbound.
```
echo 1 > rom
cat rom > /home/vm/gpu.rom
echo 0 > rom
```

When you have this ROM dumped add the following entry using `virsh edit NAME_OF_VM` under the devices section near the bottom. You need to find the correct PCI device to add the rom to. You can look for the hex identifier to figure this out. This is the XML snippet for the above Nvidia GPU.
```
 <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
      </source>
      <rom bar='on' file='/home/david/gpu.rom'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x04' function='0x0'/>
    </hostdev>
```

the bus and function values under `<source>` let me know that this is the **03:00.0** device which is my GPU.

## Contributions
Feel free to post PRs if you have anything to add. Also feel free to open an issue if you require some help or have found an error of some sorts.


