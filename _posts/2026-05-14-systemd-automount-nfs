---
title: Using systemd to Automount an NFS Share
categories: [Guides,systemd]
tags: [guides,incus,linux,debian,oci]
image:
  path: assets/img/title/systemd.svg
---

This is a reminder for me about how I have mounted my NFS shares.

When creating systemd unit files, you should put them in `/etc/systemd/system/` if they are for system-wide us. You can learn more about the loction of unit files [here](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html#Unit%20File%20Load%20Path). You can have systemd mount and automount unit files created [automatically from fstab](https://www.freedesktop.org/software/systemd/man/latest/systemd.mount.html#fstab) too, but I prefer to create the unit files myself.

With my NFS shares I place them under `/nfs` and not `/mnt`.

## Mount
First I am creating a mount unit file.

```bash
sudo vim /etc/systemd/system/nfs-share.mount
```

### Unit file contents
```
[Unit]
Description=Mount /volume/share from storage.example.net using NFS

[Mount]
What=storage.example.net:/volume/share/
Where=/nfs/share
Type=nfs
Options=defaults
TimeoutSec=5

[Install]
WantedBy=multi-user.target
```

## Automount
Next is the automount unit file.
```bash
sudo vim /etc/systemd/system/nfs-share.automount
```

### Unit file contents
```
[Unit]
Description=Automount /volume/share from storage.example.net using NFS
ConditionPathExists=/nfs/share

[Automount]
Where=/nfs/sahre
TimeoutIdleSec=10

[Install]
WantedBy=multi-user.target
```

## Enable

You will need to reload the systemd daemon

```bash
sudo systemctl daemon-reload
```

Then you need to enable and start the automount unit. You do not need to enable or start the mount unit.
```bash
sudo systemctl enable nfs-share.automount
sudo systemctl start nfs-share.automount
```

Now when you browse to `/nfs/share` it will mount on access.
