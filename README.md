# My VFIO Setup
This document describes how to recreate my personal VFIO setup.

## Hardware
- CPU: Ryzen 7 1700
- Motherboard: ASUS ROG STRIX X370-F GAMING
- RAM: 16 GB
- Host GPU: RX 570
- Guest GPU: GTX 980 Ti

## Prerequesites
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
Edit **etc/libvirt/qemu.conf** and find the commented line that starts with `nvram`. Edit these lines to reflect the path of your OVMF files. 
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





