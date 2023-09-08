---
title: Configure GPU Passthrough to LXD Container
categories: [Guides,LXD]
tags: [guides,ubuntu,linux,jellyfin,nvidia,lxd]
image: 
  path: /assets/img/title/lxd-gpu-passthrough.svg
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

After much playing I found the best way to get this working is to install using the following.

```bash
sudo apt install nvidia-headless-530
sudo apt install nvidia-utils-530
```

Restart the computer now.

### Configure Jellyfin Container

First you need to discover the PCI address of the GPU. That is done by running the following.

`sudo lshw -C display`

This will show an output like this.

```
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

You can see the PCI address under bus info for the Nvidia GPU is 0000:01:00.0. Now we can configure the jellyfin container.
The following commands add the necessary nvidia requirements to the container, add a device named nvidia-gpu and tell it to use the gpu with the PCI address of of 0000:01:00.0.
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

```
cur: 350 MHz
min: 350 MHz
RP1: 350 MHz
max: 1150 MHz
```

### Configure Jellyfin Container

This is much simpler to do than the Nvidia configuration. You should only need to pass through the GPU to the server. The difference here was the requirement to allow access by setting the GID of the video group in the container. All the containers I checked had the video group with the GID or 44. You can check by running

```bash
lxc exec jellyfin -- grep video /etc/group
```

The command is as simple as the following. Here I have used intel-gpu as the device name. You can set the name to anything you would like.

```bash
lxc config device add jellyfin intel-gpu gpu gid=44 pci=0000:00:02.0
```

The pci address comes from the `lshw` from above as used in the Nvidia configuration.

You now need to enable Intel transcoding in Jellyfin. To test that it is working, play a video, and on the host run `intel_gpu_top`. I have tested this and it works to switch between the hardware transcoding.

## Testing Passthrough With CUDA Toolkit

If you aren't using Jellyfin, this will show you how to test the passthrough using Nvidia's CUDA utilities. First, you will need to install NVIDIA CUDA toolkit.

```bash
sudo apt install nvidia-cuda-toolkit --no-install-recommends
```

> --no-install-recommends parameter will stop it from installing recommended packages that require a full GUI to be installed.
{: .prompt-info }

Now you can clone the Nvidia CUDA samples from github with the following and then change to the bandwidthTest utility sample directory.

```bash
git clone https://github.com/NVIDIA/cuda-samples.git
cd cuda-samples/Samples/1_Utilities/bandwidthTest
```

You will need to edit the Makefile to point to the correct location for the `nvcc` binary using your preferred editor. Change the file as shown below.

```diff
# Location of the CUDA Toolkit
- CUDA_PATH ?= /usr/local/cuda
+ CUDA_PATH ?= /usr
```

Now run the command `make`. I ran into an error when attempting this:

```
/usr/bin/nvcc -ccbin g++ -I../../../Common -m64 --threads 0 --std=c++11 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_80,code=sm_80 -gencode arch=compute_86,code=sm_86 -gencode arch=compute_89,code=sm_89 -gencode arch=compute_90,code=sm_90 -gencode arch=compute_90,code=compute_90 -o bandwidthTest.o -c bandwidthTest.cu
nvcc fatal   : Unsupported gpu architecture 'compute_89'
make: *** [Makefile:324: bandwidthTest.o] Error 1
```

I had to remove `89` and `90` from the make file and re-ran the `make` command.

```diff
# Gencode arguments
ifeq ($(TARGET_ARCH),$(filter $(TARGET_ARCH),armv7l aarch64 sbsa))
SMS ?= 53 61 70 72 75 80 86 87 90
else
- SMS ?= 50 52 60 61 70 75 80 86 89 90
+ SMS ?= 50 52 60 61 70 75 80 86
endif
```

Once successful in compiling you can run `./bandwidthTest` and you will see the following.

```
[CUDA Bandwidth Test] - Starting...
Running on...

 Device 0: Quadro P600
 Quick Mode

 Host to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)        Bandwidth(GB/s)
   32000000                     6.4

 Device to Host Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)        Bandwidth(GB/s)
   32000000                     6.6

 Device to Device Bandwidth, 1 Device(s)
 PINNED Memory Transfers
   Transfer Size (Bytes)        Bandwidth(GB/s)
   32000000                     52.9

Result = PASS

NOTE: The CUDA Samples are not meant for performance measurements. Results may vary when GPU Boost is enabled.
```

Now to test this, you can copy the bandwidthTest binary to your LXD container, that you have configured passthrough, and run it. Make sure to change "container" to the name of your container when running this command.

```bash
lxc file push ~/cuda-samples/Samples/1_Utilities/bandwidthTest/bandwidthTest container/root/
lxc exec container -- /root/bandwidthTest
```

If successful, you will see the same printout as above that you ran on the host.
