---
title: Configure Incus for Docker
categories: [Guides,Incus]
tags: [guides,incus,linux,lxc,docker,debian]
mermaid: true
image:
  path: assets/img/title/configure-incus-for-docker.svg
---

With this guide you will learn how to make Docker run inside an lxc container using Incus. This uses new functionality in ZFS and Incus (originally introduced in [LXD 5.17](https://github.com/canonical/lxd/releases/tag/lxd-5.17)). [ZFS 2.2](https://github.com/openzfs/zfs/releases/tag/zfs-2.2.0) introduced Linux container support for overlayfs. I have always had issues with running docker inside LXC (probably my own lack of understanding) but with this it works perfectly.

## Requirements
- You followed my guide for configuring [Incus on Debian with ZFS 2.2](/posts/configure-incus-on-debian/)

## Create Container

I am creating a container named docker using the Debian Bookworm image. This command also sets the necessary configuration for allowing nesting.

```bash
incus launch images:debian/12 docker -c security.nesting=true -c security.syscalls.intercept.mknod=true -c security.syscalls.intercept.setxattr=true
```

## Configure ZFS Pool Delegation

The delegation of ZFS is done on the volume for the container. I stop the container first because you will need to restart it afterwards anyway.

```bash
incus stop docker
incus storage volume set default container/docker zfs.delegate=true
incus start docker
```
For reference it works like `incus storage volume set <storage name> container/<container name> zfs.delegate=true`

## Install Docker in the Container

Now you can install Docker in the container and have fun with Docker.
