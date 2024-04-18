---
title: Configure Incus on Debian
categories: [Guides, Incus]
tags: [linux,debian,incus,guides,lxc]
image:
  path: /assets/img/title/configure-incus-on-debian.svg
---
> This was previously a guide for Using LXD on Ubuntu.
{: .prompt-info }

This guide is a look into how I setup my homelab. This is the first part to a group of guides I am writing.

## Requirements

- A reasonable computer
  - Recent CPU
  - 16GB memory minimum
  - 60GB drive for OS minimum
  - As big a drive as possible for container storage
- Debian 12

## Install Debian 12

You can install Desktop if you feel more comforatble having a GUI, but it's unnecessary as all my guides will be using CLI.

I am not going to go through installing Debian, as there are lots of guides around about how to install.

After install; I alway update everything and configure the timezone.

```bash
sudo apt update
sudo apt full-upgrade
sudo dpkg-reconfigure tzdata
```

## Update APT Sources

I also configure backports to install ZFS and Incus. The reason for installing ZFS from backports is that it install version 2.2 which allows to expose your zfs storage to the container. This makes nesting work nicely. You can install the current version but will be unable to delegate zfs to containers.

To do this add the following line to `/etc/apt/sources.list` and then update apt.

`deb http://deb.debian.org/debian bookworm-backports main contrib`

## Install ZFS

To install zfsutils from backports, you run the following command.

`sudo apt install zfsutils-linux/bookworm-backports`

## Install Incus

I install Incus using the [Zabbly repositories](https://github.com/zabbly/incus). Current version is 6.0 LTS.

## Initialise Incus

First make sure you know the location of the second block device for container storage. You can check with the following command.

```bash
sudo lshw -short -c disk
```

Which will output something similar to this. And in my case the block device for Incus storage is /dev/sdb.

```bash
H/W path               Device      Class          Description
=============================================================
/0/100/1f.2/0          /dev/sda    disk           120GB HARDDISK
/0/100/1f.5/0.0.0      /dev/sdb    disk           500GB HARDDISK
```

To initialise Incus you run `incus admin init`. Note that I am also setting the IPv4 address range for the bridge to 10.10.10.1/24. This will be used in other guides. Also because I am weird I set the ipv6 range to the lowest [unique local address](https://en.wikipedia.org/wiki/Unique_local_address) possible! You can select auto here for both and take note of the IPv4 address range.

> If the question for storage backend does not show it means ZFS has not been detected and you may need to restart the server.
{: .prompt-warning }

```
Would you like to use clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Name of the storage backend to use (btrfs, dir, lvm, zfs) [default=zfs]:
Create a new ZFS pool? (yes/no) [default=yes]:
Would you like to use an existing block device? (yes/no) [default=no]: yes
Path to the existing block device: /dev/sdb
Would you like to create a new local network bridge? (yes/no) [default=yes]: yes
What should the new bridge be called? [default=incusbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 10.10.10.1/24
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: fd00::1/64
Would you like the server to be available over the network? (yes/no) [default=no]: no
Would you like stale caches images to be updated automatically? (yes/no) [default=yes]:
Would you like a YAML "init" preseed to be printed? (yes/no) [default=no]:
```

Incus is now ready to start creating containers. One thing I like to do is modify the default lxc profile to set the timezone in the containers automatically. To do this you enter the following.

```bash
incus profile set default environment.TZ Region/City
```

For a list of selections see [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

We are now ready to start creating containers.

# Useful Commands

- `incus list` to view a list of containers.
- `incus config show <container name>` to view the containers configuration.
- `incus admin init --dump` to view the intial configuration for Incus.
- `incus exec <containter name> bash` to connect a container terminal session using bash.
