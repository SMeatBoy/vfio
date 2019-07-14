# My VFIO Setup
This document describes how to recreate my personal VFIO setup.

## Hardware
- CPU: Ryzen 7 1700
- Motherboard: ASUS ROG STRIX X370-F GAMING
- RAM: 16 GB
- Host GPU: RX 570
- Guest GPU: GTX 980 Ti

## Prerequesites
#### Set Kernel Parameters in /etc/default/grub
Before:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
After:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on iommpu=pt video=efifb:off vfio-pci.ids=10de:17c8,10de:0fb0 isolcpus=8-15 nohz_full=8-15 rcu_nocbs=8-15"
```
- `amd_iommu=on`: Enable IOMMU
- `iommu=pt`: Improves performance. See [this](http://mails.dpdk.org/archives/dev/2014-October/007411.html) for a (very) detailed description.
- `video=efifb:off`: No display output from guest GPU without this. See [this post on The Passthrough POST](https://passthroughpo.st/explaining-csm-efifboff-setting-boot-gpu-manually/)
- `isolcpus=8-15`: Isolates and pins CPU cores so that they are exclusively used by the guest OS. **Caution:** Disables host OS to use these CPU cores.
- `nohz_full=8-15` and `rcu_nocbs=8-15`: Removes microstutters in the guest OS. I don't know exactly why this works. See [this old Reddit post](https://www.removeddit.com/r/VFIO/comments/6vgtpx/high_dpc_latency_and_audio_stuttering_on_windows/dm0sfto/).
