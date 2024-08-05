## Intel Core Graphics Passthrough ROM

### Introduction
- This ROM is an Intel 10-13 Core Graphics Passthrough PCI optionROM. It can be used with OVMF to start the virtual machine, and the display HDMI/DP output screen and HDMI/DP sound work normally.
- This ROM is easy to use, and there is no need to modify or customize OVMF. Just use PVE!
- The virtual machine starts without a distorted screen or a blue screen.

### How to use:

+ This ROM needs to use two rom files:
  - Core display pass-through OptionROM: gen12_igd.rom -- basically universal for all platforms
  - GOP ROM: --- Select the corresponding rom file according to different core display platforms, see the table below:

GOP ROM file name     | Applicable CPU platform
----------------------|----------------------------
gen12_gop.rom         | Intel 11-13th generation
5105_gop.rom          | N5105 / N5095
8505_gop.rom          | 8505

  - Please select the corresponding GOP ROM according to the CPU of the host. Selecting the wrong GOP ROM function will result in no startup screen

+ Copy these two rom files to /use/share/kvm/
+ Because two rom files are used, in the conf configuration file, one rom file is added to the graphics card and the other is added to the sound card, please pay attention.
```
hostpci0: 0000:00:02.0,legacy-igd=1,romfile=gen12_igd.rom
hostpci1: 0000:00:1f.3,romfile=gen12_gop.rom
```
### You can refer to my 100.conf, pay attention to the following matters:

+ The model must be i440fx, (QEMU does not support Q35 display in Legacy mode, QEMU can be customized to support Q35, which is not discussed in this article)
+ BIOS must be OVMF, Intel core graphics no longer supports traditional BIOS boot
+ Add legacy-igd=1 to the core graphics PCI to support display in Legacy mode
+ Add args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on
+ "-debugcon in args file:/root/d-debug.log -global isa-debugcon.iobase=0x402” is a debugging file, don’t add it if you mind
+ The virtual machine memory is at least 4G, less than 4G may have problems
+ It is recommended to x-igd-gms=0x2, and pay attention to the BIOS setting: DVMT pre allocated, not more than 64M

### Usage restrictions

+ This ROM does not support commercial use, only for DIY enthusiasts technical research
+ This ROM only supports Intel core graphics, not AMD
+ Only supports UEFI, normal boot. Secure boot is not supported
+ Only supports OVMF mode, seabios does not support
+ The memory is at least 4G, less than 4G may have problems
+ Note the BIOS setting: DVMT pre allocated, not more than 64M, 64M corresponds to x-igd-gms=0x2, if it exceeds 64M, x-igd-gms needs to be increased!

### This ROM is only tested in the following environments, and I have not tested other environments.

+ South China Gold 760 motherboard + 13600CPU
+ PVE 8.0.3
+ The test results are basically perfect, with logo and startup screen, no screen distortion, and HDMI/DP sound working properly. You can also complete the entire Windows reinstallation.

#### YouTube plays 4K video: Task Manager GPU occupancy
> ![GPU](https://raw.githubusercontent.com/gangqizai/igd/main/test_screenshot/task_manager.PNG "GPU")

#### HDMI Audio 
> ![HDMI Audio](https://raw.githubusercontent.com/gangqizai/igd/main/test_screenshot/hdmi-audio.PNG "HDMI Audio")

### Common errors

1. No startup logo display and boot animation
  + PVE shell Use the command to start the virtual machine. You must only see the following sound card errors. If there are other errors, check the PVE passthrough settings. Refer to the following personal settings
```
    root@pve:/etc/pve/qemu-server# qm start 300
    kvm: vfio: Cannot reset device 0000:00:1f.3, no available reset mechanism.
    kvm: vfio: Cannot reset device 0000:00:1f.3, no available reset mechanism.
```
  + /etc/modprobe.d/vifo.conf file cannot have disable_vga=1. If there is one, delete it! Then run the command "update-initramfs -u" and restart PVE

2. No HDMI/DP sound
  + First check the audio controller driver: Device Manager->System Devices->HD audio controller. If not, check the chipset and audio driver
  + ![HD Audio controller](https://raw.githubusercontent.com/gangqizai/igd/main/test_screenshot/hdmi-audio-controller.PNG "HD Audio Controller")
  + You can delete the graphics driver and install it after restarting
  + Install Windows completely from scratch in the core graphics pass-through mode (this ROM supports it, and there is no screen distortion during the installation process)

### Welcome to provide debugging information

+ I only have one machine and cannot test more platforms. Welcome to provide test debugging information.
+ Confirm to open args in the conf file: -debugcon file:/root/igd_debug.log -global isa-debugcon.iobase=0x402
+ Provide the generated debug file: /root/igd_debug.log
+ In the PVE host shell, send two commands: lspci -s 00:02.0 -xxx and lspci -s 00:00.0 -xxx, and send the output results to me

#### If you don’t want to use two rom files, you can also combine them into one.

### PVE graphics card pass-through settings:

- Edit grub, add intel_iommu=on
```
nano /etc/default/grub
```
- GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
update-grub
```
- Edit /etc/modules
```
nano /etc/modules
```
- Add the following modules
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
- Add the graphics card driver to the blacklist
```
echo "blacklist i915" >> /etc/modprobe.d/pve-blacklist.conf
```
- Bind vfio-pci through the device ID and execute lspci -n | grep -E "0300" to view and record the integrated graphics VendorID and DeviceID
```
echo "options vfio-pci ids=8086:a780" >> /etc/modprobe.d/vifo.conf
```
```
update-initramfs -u
reboot
```
#### vifo.conf does not have disable_vga=1, delete it if it exists!

### PVE Windows 10 VM example conf:
```
args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on -debugcon file:/root/igd_debug.log -global isa-debugcon.iobase=0x402
bios: ovmf
boot: order=scsi0;ide0
cores: 4
cpu: host
efidisk0: local-lvm:vm-200-disk-0,efitype=4m,size=4M
hostpci0: 0000:00:02.0,legacy-igd=1,romfile=gen12_igd.rom
hostpci1: 0000:00:1f.3,romfile=gen12_gop.rom
ide0: local:iso/virtio-win-0.1.229.iso,media=cdrom,size=522284K
ide2: local:iso/Win10_22H2_English_x64.iso,media=cdrom,size=5971862K
machine: pc-i440fx-8.0
memory: 8192
meta: creation-qemu=8.0.2,ctime=1692935 943
name: Win10
net0: virtio=32:02:47:A8:40:01,bridge=vmbr0,firewall=1
numa: 0
ostype: win10
scsi0: local-lvm:vm-200-disk-1,iothread=1,size=60G
scsihw: virtio-scsi-single
serial1: socket
smbios1: uuid=d9fadf8d-eae0-4cb3-a3d5-cf222c305b91
sockets: 1
usb0: host=046d:c016,usb3=1
usb1: host=1c4f:0059,usb3=1
vga: none
vmgenid: 8f84e0f9-534d-440b-bb6 d-09ed75cdc167
```
Email: markc@renta.net


Email: gangqizai@gmail.com
