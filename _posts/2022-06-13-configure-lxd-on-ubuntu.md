---
title: Configure LXD on Ubuntu
categories: [guides, lxd]
tags: [linux, ubuntu, lxd, guide, lxc]
---

# Introduction

This guide is a look into how I setup my homelab. This is the first part to a group of guides I am writing.

## Requirements

- A reasonable computer
  - Recent CPU
  - 16GB memory minimum
  - 60GB drive for OS minimum
  - As big a drive as possible for container storage
- Ubuntu 22.04

# Install Ubuntu Server

You can install Ubuntu Desktop if you feel more comforatble having a GUI, but it's unnecessary as all my guides will be using CLI.

I am not going to go through installing Ubuntu Server, as there are lots of guides around about how to install Ubuntu.

After install; I alway update everything and configure timezone correctly.

```bash
sudo apt update
sudo apt full-upgrade
sudo dpkg-reconfigure tzdata
```

# Initialise LXD

First make sure you know the location of the second block device for container storage. You can check with the following command.

```bash
sudo lshw -short -c disk
```

Which will output something similar to this. And in my case the block device for LXD storage is /dev/sdb.

```bash
H/W path               Device      Class          Description
=============================================================
/0/100/1f.2/0          /dev/sda    disk           120GB HARDDISK
/0/100/1f.5/0.0.0      /dev/sdb    disk           500GB HARDDISK
```

To configure LXD you run `lxd init`. Note that I am also setting the IPv4 address range for the bridge to 10.10.10.1/24. This will be used in other guides. Also because I am weird I set the ipv6 range to the lowest [unique local address](https://en.wikipedia.org/wiki/Unique_local_address) possible! You can select auto here for both and take note of the IPv4 address range.

```
Would you like to use LXD clustering? (yes/no) [default=no]: no
Do you want to configure a new storage pool? (yes/no) [default=yes]: yes
Name of the new storage pool [default=default]: default
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]: zfs
Create a new ZFS pool? (yes/no) [default=yes]: yes
Would you like to use an existing block device? (yes/no) [default=no]: yes
Path to the existing block device: /dev/sdb
Would you like to connect to a MAAS server? (yes/no) [default=no]: no
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 10.10.10.1/24
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: fd00::1/64
Would you like LXD to be available over the network? (yes/no) [default=no]: no
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: no
```

LXD is now ready to start creating containers. One thing I like to do is modify the default lxc profile to set the timezone in the containers automatically. To do this you enter the following.

```bash
lxc profile set default environment.TZ Region/City
```

For a list of selections see [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

We are now ready to start creating containers.

# Useful Commands

- `lxc list` to view a list of containers.
- `lxc config show <container name>` to view teh containers configuration.
- `lxd init --dump` to view the intial configuration for LXD.
- `lxc exec <containter name> bash` to connect a container terminal session using bash
