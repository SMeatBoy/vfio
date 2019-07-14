# Yet Another VFIO Guide

This guide describes how to create my personal VFIO setup.

## Table of Contents
- [Hardware](https://github.com/SMeatBoy/vfio/blob/master/README.md#hardware)
- [System Configuration](https://github.com/SMeatBoy/vfio/blob/master/README.md#system-configuration)
  - [Set Kernel Parameters](https://github.com/SMeatBoy/vfio/blob/master/README.md#set-kernel-parameters)
  - [Blacklist GPU to prevent driver loading](https://github.com/SMeatBoy/vfio/blob/master/README.md#blacklist-gpu-to-prevent-driver-loading)
  - [Load VFIO kernel modules](https://github.com/SMeatBoy/vfio/blob/master/README.md#load-vfio-kernel-modules)
  - [Update GRUB and Initramfs](https://github.com/SMeatBoy/vfio/blob/master/README.md#update-grub-and-initramfs)
- [Set up QEMU](https://github.com/SMeatBoy/vfio/blob/master/README.md#set-up-qemu)
  - [Install virtualization packages](https://github.com/SMeatBoy/vfio/blob/master/README.md#install-virtualization-packages)
  - [Set path of OVMF firmware](https://github.com/SMeatBoy/vfio/blob/master/README.md#set-path-of-ovmf-firmware)
  - [*Optional: Change user for QEMU*](https://github.com/SMeatBoy/vfio/blob/master/README.md#optional-change-user-for-qemu-)
- [Create and configure the VM](https://github.com/SMeatBoy/vfio/blob/master/README.md#create-and-configure-the-vm)
  - [Create VM in virt-manager](https://github.com/SMeatBoy/vfio/blob/master/README.md#create-vm-in-virt-manager)
    - [Initial VM creation](https://github.com/SMeatBoy/vfio/blob/master/README.md#initial-vm-creation)
    - [Basic configuration](https://github.com/SMeatBoy/vfio/blob/master/README.md#basic-configuration)
  - [Advanced configuration with virsh](https://github.com/SMeatBoy/vfio/blob/master/README.md#advanced-configuration-with-virsh)
    - [XML Namespace](https://github.com/SMeatBoy/vfio/blob/master/README.md#xml-namespace)
    - [CPU configuration](https://github.com/SMeatBoy/vfio/blob/master/README.md#cpu-configuration)
    - [Fix Error 43](https://github.com/SMeatBoy/vfio/blob/master/README.md#fix-error-43)
- [*Optional: Evdev Passthrough*](https://github.com/SMeatBoy/vfio/blob/master/README.md#optional-evdev-passthrough)
  - [Finding the correct input devices](https://github.com/SMeatBoy/vfio/blob/master/README.md#finding-the-correct-input-devices)
  - [Add devices to your VM](https://github.com/SMeatBoy/vfio/blob/master/README.md#add-devices-to-your-vm)
  - [Add files to QEMU](https://github.com/SMeatBoy/vfio/blob/master/README.md#add-files-to-qemu)
- [Credits/Sources](https://github.com/SMeatBoy/vfio/blob/master/README.md#creditssources)
## Hardware
- CPU: Ryzen 7 1700
- Motherboard: ASUS ROG STRIX X370-F GAMING
- RAM: 16 GB
- Host GPU: RX 570
- Guest GPU: GTX 980 Ti

## System Configuration
#### Set Kernel Parameters 
Edit **/etc/default/grub**
###### Before:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
###### After:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on iommpu=pt video=efifb:off vfio-pci.ids=10de:17c8,10de:0fb0 isolcpus=8-15 nohz_full=8-15 rcu_nocbs=8-15"
```
- `amd_iommu=on`: Enable IOMMU
- `iommu=pt`: Improves performance. See [this](http://mails.dpdk.org/archives/dev/2014-October/007411.html) for a (very) detailed description.
- `video=efifb:off`: No display output from guest GPU without this. See [this post on The Passthrough POST](https://passthroughpo.st/explaining-csm-efifboff-setting-boot-gpu-manually/)
- `vfio-pci.ids=10de:17c8,10de:0fb0`: Use vfio-pci driver for guest GPU
- `isolcpus=8-15`: Isolates and pins CPU cores so that they are exclusively used by the guest OS. **Caution:** Disables host OS to use these CPU cores
- `nohz_full=8-15` and `rcu_nocbs=8-15`: Removes microstutters in the guest OS. I don't know exactly why this works. See [this old Reddit post](https://www.removeddit.com/r/VFIO/comments/6vgtpx/high_dpc_latency_and_audio_stuttering_on_windows/dm0sfto/)

#### Blacklist GPU to prevent driver loading
Create **/etc/modprobe.d/vfio.conf** and add these parameters:

```
  options vfio-pci ids=10de:17c8,10de:0fb0 
  options kvm ignore_msrs=1
```
- `options vfio-pci ids=10de:17c8,10de:0fb0`: Further blacklisting of guest GPU
- `options kvm ignore_msrs=1`: Windows will not boot without this. See [this post on the Level1Techs forum](https://forum.level1techs.com/t/windows-10-1803-as-guest-with-qemu-kvm-bsod-under-install/127425/13)

#### Load VFIO kernel modules
Edit **/etc/initramfs-tools/modules** and append these parameters:
```
vfio_pci
vfio
vfio_iommu_type1
vfio_virqfd
```
#### Update GRUB and Initramfs
Next, update GRUB and Initramfs to feature all the changes made above by running `update grub && update-initramfs -u`

## Set up QEMU
#### Install virtualization packages
Run `apt install qemu qemu-kvm libvirt0 ovmf virt-manager` to install packages needed for virtualization.
#### Set path of OVMF firmware
Edit **etc/libvirt/qemu.conf** and find the commented line that starts with `nvram`. Edit these lines to reflect the path of your OVMF files. Always restart libvirtd after editing by executing `systemctl restart libvirtd.service`
###### Before
```
#nvram = [
#   "/usr/share/OVMF/OVMF_CODE.fd:/usr/share/OVMF/OVMF_VARS.fd",
#   "/usr/share/OVMF/OVMF_CODE.secboot.fd:/usr/share/OVMF/OVMF_VARS.fd",
#   "/usr/share/AAVMF/AAVMF_CODE.fd:/usr/share/AAVMF/AAVMF_VARS.fd",
#   "/usr/share/AAVMF/AAVMF32_CODE.fd:/usr/share/AAVMF/AAVMF32_VARS.fd",
#   "/usr/share/OVMF/OVMF_CODE.ms.fd:/usr/share/OVMF/OVMF_VARS.ms.fd"
#]
```
###### After
```
nvram = [
   "/usr/share/OVMF/OVMF_CODE.fd:/usr/share/OVMF/OVMF_VARS.fd",
]
#   "/usr/share/OVMF/OVMF_CODE.secboot.fd:/usr/share/OVMF/OVMF_VARS.fd",
#   "/usr/share/AAVMF/AAVMF_CODE.fd:/usr/share/AAVMF/AAVMF_VARS.fd",
#   "/usr/share/AAVMF/AAVMF32_CODE.fd:/usr/share/AAVMF/AAVMF32_VARS.fd",
#   "/usr/share/OVMF/OVMF_CODE.ms.fd:/usr/share/OVMF/OVMF_VARS.ms.fd"
```
#### Optional: Change user for QEMU
This is only needed if evdev passthrough will be used. 
As I understand it is best practice to execute QEMU as a non-login user created for this purpose. One may also change this value to the own user or root if so preferred. 

First, create a new user by executing `useradd -s /usr/sbin/nologin -r -M -d /dev/null vfio`. 
Then, edit **etc/libvirt/qemu.conf** once again. The relevant line is around 440.
###### Before:
```
#user = root 
```
###### After:
```
user = vfio
```

## Create and configure the VM
This can be split into two parts. In the first part a virtual machine will be created with virt-manager. The second part configures the VM to improve performance and (most importantly) work around the well-known *Error 43*.

#### Create VM in virt-manager
##### Initial VM creation
Open virt-manager and create a new virtual machine. A window should open. 
- Step 1: Choose local install media
- Step 2: Select you install media. You might have to create a storage pool first. In this case choose filesystem directory.
- Step 3: Select 8144 and 8 for memory and CPU respectively.
- Step 4: Depending on your setup you can choose to
  - use the default disk image
  - create a disk image in a place of your choosing
  - specify a whole disk as storage location, e.g. /dev/sdb3
- Step 5: **Important:** Tick *Customize configuration before install*

##### Basic configuration
A couple of steps need to be done in the new configuration overview window. 
- Choose the correct firmware in the *Overview* tab. Switch from BIOS to UEFI. The UEFI entry should feature the previously specified nvram path. 
- Change the CPU configuration in the *CPUs* tab. 
  - Deselect *Copy host CPU configuration* and set the model to `host-passthrough`. 
  - Select *Manually set CPU topology*. In this case 1 socket, 4 cores and 2 threads (per core).
- Change the Disk Bus to VirtIO in the *SATA Disk 1* tab
- Optional: Remove the *Tablet*, *Display Spice* and *Video QXL* devices. We're doing GPU-passthrough after all. 
- Really Optional: Remove the *Sound ich9* device. I just can't get guest to host audio passthrough to work. Your mileage may vary. 
- Add Hardware: Select all devices that you want to use inside your VM.  
  - Select both Guest GPU devices under PCI Host device
  - *Optional: Select a USB controller or HD audio device under PCI Host device*
  - *Optional: Select single USB controller*
  - *Optional: Select additional storage and/or further devices. Just remember that you need some kind of input device to install windows if you aren't using evdev passthrough.*

Hit *Begin Installation* once you configured the VM to your liking. Kill the VM before installing windows since your new VM is not fully configured. 

#### Advanced configuration with virsh
The following steps are performed inside the VM's XML file. 
Edit by executing `virsh edit VM_NAME`.
##### XML Namespace
Declare the XML namespace in the first line
###### Before:
```<domain type='kvm'>```
###### After:
```<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>```
##### CPU configuration
Change the CPU configuration to reflect the isolated cpu cores and number of cores of your VM. 
###### Before:
```
<vcpu placement='static' current='1'>8</vcpu>
```
###### After:
```
<vcpu placement='static'>8</vcpu>
  <cputune>
    <vcpupin vcpu='0' cpuset='8'/>
    <vcpupin vcpu='1' cpuset='9'/>
    <vcpupin vcpu='2' cpuset='10'/>
    <vcpupin vcpu='3' cpuset='11'/>
    <vcpupin vcpu='4' cpuset='12'/>
    <vcpupin vcpu='5' cpuset='13'/>
    <vcpupin vcpu='6' cpuset='14'/>
    <vcpupin vcpu='7' cpuset='15'/>
    <emulatorpin cpuset='0-1'/>
  </cputune>
```
#### Fix Error 43
Add a fake vendor_id and hide KVM in the features section.
###### Before:
```
<features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
    </hyperv>
    <vmport state='off'/>
  </features>
```
###### After:
```
<features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vendor_id state='on' value='1234567890ab'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>
```
## *Optional: Evdev Passthrough*
Evdev passthrough works like a virtual USB/KVM switch. Clicking both CTRLs on your keyboard switches your input devices between guest and host. The Passthrough POST has a really [nice tutorial](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/) explaining the matter much better.  

I did not end up using it (even though I would have really liked to). Sometimes when switching from guest to host audio in the Windows guest hangs until i disable and enable the audio device (HDMI audio from 980 Ti). Instead I use [Synergy](https://symless.com/synergy) to switch mouse and keyboard between guest and host. 

#### Finding the correct input devices
Your mouse and keyboard actions (in Linux fashion) are represented as a file under **/dev/input**. Try to find the event files that correspondent to the actual key events by first listing the content of **/dev/input/by-id**.
###### Example:
```
$ ls -l /dev/input/by-id

total 0
lrwxrwxrwx 1 root root  9 Jul 14 18:28 usb-Logitech_Gaming_Mouse_G502_0C65344F3231-event-if01 -> ../event4
lrwxrwxrwx 1 root root  9 Jul 14 18:28 usb-Logitech_Gaming_Mouse_G502_0C65344F3231-event-mouse -> ../event2
lrwxrwxrwx 1 root root  9 Jul 14 18:28 usb-Logitech_Gaming_Mouse_G502_0C65344F3231-if01-event-kbd -> ../event3
lrwxrwxrwx 1 root root  9 Jul 14 18:28 usb-Logitech_Gaming_Mouse_G502_0C65344F3231-mouse -> ../mouse0
lrwxrwxrwx 1 root root 10 Jul 14 18:28 usb-Logitech_Logitech_G710_Keyboard-event-if01 -> ../event12
lrwxrwxrwx 1 root root 10 Jul 14 18:28 usb-Logitech_Logitech_G710_Keyboard-event-kbd -> ../event10
lrwxrwxrwx 1 root root 10 Jul 14 18:28 usb-Logitech_Logitech_G710_Keyboard-if01-event-kbd -> ../event11
```
Then, `cat` devices that sound plausible. In my case all of the above... If the output is random gibberish you have found one of you input devices. Repeat until all devices have been found. Note the event file in /dev/input that the file symlinks to. You will need that path (e.g. /dev/input/event10/) for later configuration
###### Example:
```
$ cat /dev/input/by-id/usb-Logitech_Logitech_G710_Keyboard-event-kbd

+]�`d-k+]Pz-k+]Pz -k+]Pz.k+]��.k+]��.k+]��2/k+]�# /k+]�#/k+]�#3/k+]�/k+]�/k+]�/k+]�!/k+]�/k+]�4/k+]�� /k+]��/k+]��/k+]%�!/k+]%�/k+]%�7k+]I<
```
#### Add devices to your VM
Add all devices found the previous step to your VM by editing its XML near the bottom of the file via `virsh edit VM_NAME`. Add `grab_all=on,repeat=on` to your keyboard.

###### Before:
```
  </devices>
</domain>
```
###### After:
```
  </devices>
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/event2'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse2,evdev=/dev/input/event3'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/event10,grab_all=on,repeat=on'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd2,evdev=/dev/input/event12'/>
  </qemu:commandline>
</domain>
```
#### Add files to QEMU
Add the same devices/files to the *cgroup_devices_acl* to your QEMU config in **etc/libvirt/qemu.conf/**
The relevant section should be around line 480.

###### Before:
```
# cgroup_device_acl = [
#    "/dev/null", "/dev/full", "/dev/zero",
#    "/dev/random", "/dev/urandom",
#    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
#    "/dev/rtc","/dev/hpet", "/dev/sev"
#
# ]
```

###### After:
```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet", "/dev/sev",
    "/dev/input/event2",
    "/dev/input/event3",
    "/dev/input/event10",
    "/dev/input/event12",
]
```
Restart libvirtd with `systemctl restart libvirtd.service`.

#### Add exceptions to AppArmor
QEMU won't be able to access these devices if they aren't whitelisted. Therefore you have to add them as exception in **/etc/apparmor.d/abstractions/libvirt-qemu**. Add the following lines and restart AppArmor with `systemctl restart apparmor.service`:
```
  /dev/input/event2 rw,
  /dev/input/event3 rw,
  /dev/input/event10 rw,
  /dev/input/event12 rw,
```
Finally, add the previously created user *vfio* to the *input* group: `usermod -a -G input vfio`


## Credits/Sources
- Arch Wiki 
  - [(PCI passthrough via OVMF)](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#With_vfio-pci_built_into_the_kernel)
- The Passthrough POST 
  - [Explaining CSM, efifb=off, and Setting the Boot GPU Manually](https://passthroughpo.st/explaining-csm-efifboff-setting-boot-gpu-manually/)
  - [Evdev Passthrough Explained — Cheap, Seamless VM Input](https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/)
- Reddit 
  - [High DPC Latency and Audio Stuttering on Windows 10](https://www.removeddit.com/r/VFIO/comments/6vgtpx/high_dpc_latency_and_audio_stuttering_on_windows/dm0sfto/)
  - [Trouble passing evdev to guest: Could not open /dev/input/by-id/...: Permission denied](https://www.reddit.com/r/VFIO/comments/7iqw6h/trouble_passing_evdev_to_guest_could_not_open/)
- Level1Techs 
  - [Windows 10 1803 as guest with QEMU/KVM, BSOD under install](https://forum.level1techs.com/t/windows-10-1803-as-guest-with-qemu-kvm-bsod-under-install/127425/13)
- Many more I can't remember
