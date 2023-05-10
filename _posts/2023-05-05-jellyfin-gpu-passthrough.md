---
title: Configure GPU Passthrough to LXD Container
categories: [Guides,LXD]
tags: [guides,ubuntu,linux,jellyfin,nvidia,lxd]
---

I recently got hold of a mini PC with a Nvidia GPU and wanted to get hardware transcoding working with Jellyfin in an LXD container. I have previously done this on my existing mini PC that only has an Intel iGPU and I will show how to do that also.

> Make sure you disable Secure Boot in UEFI, as I had issues with having it enabled.
{: .prompt-info }

## Requirements

- Ubuntu 22.04
- Supported Nvidia GPU or Intel iGPU
- [Jellyfin](https://jellyfin.org/) running in an LXD container

## Nvidia Configuration
### Install Nvidia Drivers & Tools

After much playing I found the best wat to get this working is to install using the following.

```bash
sudo apt install nvidia-headless-530
sudo apt install nvidia-utils-530
```

Restart the computer now.

### Configure Jellyfin Container

First you need to discover the pci address of the GPU. That is done by running the following.

`sudo lshw -C display`

This will show an output like this.

```bash
  *-display
       description: VGA compatible controller
       product: GP107GL [Quadro P600]
       vendor: NVIDIA Corporation
       physical id: 0
       bus info: pci@0000:01:00.0
       version: a1
       width: 64 bits
       clock: 33MHz
       capabilities: pm msi pciexpress vga_controller bus_master cap_list rom
       configuration: driver=nvidia latency=0
       resources: irq:136 memory:f6000000-f6ffffff memory:e0000000-efffffff memory:f0000000-f1ffffff ioport:e000(size=128) memory:c0000-dffff
  *-display
       description: Display controller
       product: HD Graphics 630
       vendor: Intel Corporation
       physical id: 2
       bus info: pci@0000:00:02.0
       logical name: /dev/fb0
       version: 04
       width: 64 bits
       clock: 33MHz
       capabilities: pciexpress msi pm bus_master cap_list fb
       configuration: depth=32 driver=i915 latency=0 resolution=1920,1080
       resources: irq:135 memory:f5000000-f5ffffff memory:d0000000-dfffffff ioport:f000(size=64)
```

You can see the PCI address under bus info for the Nvidia GPU is 0000:01:00.0. Now we con configure the jellyfin container.
The following commands add the necessary nvidia requirements to the container, add a device named nvidia-gpu and tell it to using the gpu with the id of 0.
For more information about GPU devices, you can see the [LXD documentation](https://linuxcontainers.org/lxd/docs/latest/reference/devices_gpu/).

```bash
lxc config set jellyfin nvidia.runtime=true nvidia.driver.capabilities=all
lxc config device add jellyfin nvidia-gpu gpu pci=0000:01:00.0
lxc restart jellyfin
```

At this point you should only need to sign into [Jellyfin and enable hardware transcoding for Nvidia](https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia).

You can now test that it is working by running `watch -d -n 0.5 nvidia-smi` on the LXD host and playing a video that requires transcoding. You will see a process at the bottom running.

## Intel Configuration
### Install Intel GPU Tools

All you need to do is run

```bash
sudo apt install intel-gpu-tools
```

You can check this is working by running the command `sudo intel_gpu_frequency` and you will see a printout, like below, if successful.

```bash
cur: 350 MHz
min: 350 MHz
RP1: 350 MHz
max: 1150 MHz
```

### Configure Jellyfin Container

This is much simpler to do than the Nvidia configuration. You should only need to pass through the gpu to the server. The difference here was the requirement to allow access by setting the GID of the video group in the container. All the containers I checked had the video group with the GID or 44. You can check by running

```bash
lxc exec jellyfin -- grep video /etc/group
```

The command is as simple as the following. Here I have used intel-gpu as the device name. You can set the name to anything you would like.

```bash
lxc config device add jellyfin intel-gpu gpu gid=44 pci=0000:00:02.0
```

The pci address comes from the `lshw` from above as used in the Nvidia configuration.

You now need to enable Intel transcoding in Jellyfin. To test that it is working, play a video, and on the host run `intel_gpu_top`. I have tested this and it works to switch between the hardware transcoding.
