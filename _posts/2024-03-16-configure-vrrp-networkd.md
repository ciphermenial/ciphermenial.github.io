---
title: Configure VRRP with FRRouting & networkd
categories: [Guides,Networking]
tags: [guides,frrouting,linux,vrrp,networkd,systemd,incus]
mermaid: true
image:
  path: assets/img/title/configure-vrrp-networkd.svg
---

I decided that I wanted to configure my own router without using something like OPNsense. I wanted to do it with iproute2, nftables, frrouting, unbound, and kea. Kea was something I only discovered recently and wanted to give it a go. I decided I would learn how to do this using LXC containers and Incus on Debian. I want to move away from Canonical as much as possible after [LXD was taken out of Linux Containers](https://discuss.linuxcontainers.org/t/lxd-is-no-longer-part-of-the-linux-containers-project/17593).

I have ended up playing around with VRRP on FRRouting and found nothing online on how to do the network configuration side with networkd. I don't know why I ended up going down this path but thought I should write up how I made it work. The [FRRouting documentation](https://docs.frrouting.org/en/latest/vrrp.html#system-configuration) for configuring the network only gives examples for ifupdown2 or iproute2.

## Presumptions

- Debian is already installed.
- 2 NICs on each server.
- You know how to use vim.

## Enabling networkd

First you need to enable networkd, which is simply enabling and starting it with `systemctl`.

```bash
systemctl enable systemd-networkd.service
systemctl start systemd-networkd.service
```

## Configuring networkd

Since I am using Incus containers, the NICs are named eth0 and eth1. The IP addresses I am using are as follows.

#### Server 1
- eth0 ipv4: `10.10.10.3`
- eth0 ipv6: `fd00::3`
- eth1 ipv4: `10.10.20.3`
- eth1 ipv6: `fd01::3`

#### Server 2
- eth0 ipv4: `10.10.10.4`
- eth0 ipv6: `fd00::4`
- eth1 ipv4: `10.10.20.4`
- eth1 ipv6: `fd01::4`

#### VRRP
- vrrp4: 10.10.20.2
- vrrp6: fd01::2

### Configure eth0

`vim /etc/systemd/network/eth0.network`

#### Server 1

```ini
[Match]
Name=eth0

[Network]
Address=10.10.10.3/24
Gateway=10.10.10.1
DNS=1.1.1.1
DNS=1.0.0.1
IPv6AcceptRA=false
Address=fd00::3/64
Gateway=fd00::1
DNS=2606:4700:4700::1111
DNS=2606:4700:4700::1111
```

#### Server 2

```ini
[Match]
Name=eth0

[Network]
Address=10.10.10.4/24
Gateway=10.10.10.1
DNS=1.1.1.1
DNS=1.0.0.1
IPv6AcceptRA=false
Address=fd00::4/64
Gateway=fd00::1
DNS=2606:4700:4700::1111
DNS=2606:4700:4700::1111
```

### Configure eth1

`vim /etc/systemd/network/eth1.network`

#### Server 1

```ini
[Match]
Name=eth1

[Network]
Address=10.10.20.3/24
Gateway=10.10.10.3
Address=fd01::3/64
Gateway=fd00::3
MACVLAN=vrrp4
MACVLAN=vrrp6
```

#### Server 2

```ini
[Match]
Name=eth1

[Network]
Address=10.10.20.4/24
Gateway=10.10.10.4
Address=fd01::4/64
Gateway=fd00::4
MACVLAN=vrrp4
MACVLAN=vrrp6
```

### Configure MACVLAN Device
These settings are identical on each router.

#### IPv4

`vim /etc/systemd/network/vrrp4.netdev`

```ini
[NetDev]
Name=vrrp4
Kind=macvlan
MACAddress=00:00:5e:00:01:05

[MACVLAN]
Mode=bridge
```

#### IPv6

`vim /etc/systemd/network/vrrp6.netdev`

```ini
[NetDev]
Name=vrrp6
Kind=macvlan
MACAddress=00:00:5e:00:02:05

[MACVLAN]
Mode=bridge
```

### Configure MACVLAN network

#### IPv4

`vim /etc/systemd/network/vrrp4.network`

```ini
[Match]
Name=vrrp4

[Network]
IPForward=yes
Address=10.10.20.2/24
IPv6LinkLocalAddressGenerationMode=random
```

### IPv6

`vim /etc/systemd/network/vrrp6.network`

```ini
[Match]
Name=vrrp6

[Network]
IPForward=yes
Address=fd01::2/64
IPv6LinkLocalAddressGenerationMode=random
```

## Finalise networkd Configuration

You will now need to restart networkd.

`systemctl restart networkd`

If you run `ip address` you should now see the following.

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: vrrp6@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:00:5e:00:02:05 brd ff:ff:ff:ff:ff:ff
    inet6 fd01::2/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::b868:88af:2e81:a44c/64 scope link stable-privacy
       valid_lft forever preferred_lft forever
3: vrrp4@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:00:5e:00:01:05 brd ff:ff:ff:ff:ff:ff
    inet 10.10.20.2/24 brd 10.10.20.255 scope global vrrp4
       valid_lft forever preferred_lft forever
    inet6 fd01::200:5eff:fe00:105/64 scope global dynamic mngtmpaddr noprefixroute
       valid_lft 2591992sec preferred_lft 604792sec
    inet6 fe80::61f4:c2d8:9744:192e/64 scope link stable-privacy
       valid_lft forever preferred_lft forever
55: eth0@if56: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:53:b9:5f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.10.3/24 brd 10.10.10.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fd00::3/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fe53:b95f/64 scope link
       valid_lft forever preferred_lft forever
57: eth1@if58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:a8:3d:0c brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.20.3/24 brd 10.10.20.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fd01::3/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fea8:3d0c/64 scope link
       valid_lft forever preferred_lft forever
```

All you need to make sure of is that the `inet6` link local addresses for the vrrp devices don't use EUI-64. They should be random. This is discussed in the FRRouting documentation.

> Finally, take note of the generated IPv6 link local address on the interface. For interfaces on which VRRP will operate in IPv6 mode, this link local cannot be derived using the usual EUI-64 method. This is because VRRP advertisements are sent from the link local address of this interface, and VRRP uses the source address of received advertisements as part of its election algorithm.

## Configuring FRRouting

> This configuration is identical on each server.
{: .prompt-info }

First you will need to install FRRouting with apt.

`apt install frr`

Then you will need to enable the VRRP daemon.

`vim /etc/frr/daemons`

You will need to set `vrrpd=no` to `vrrpd=yes`
Now you can enable and start FRRouting.

```bash
systemctl enable frr.service
systemcrl start frr.service
```

At this point you can access the VTY shell, which is similar to using a Cisco CLI, with the command `vtysh`.

 You will see the following and you are now in the shell. In this example my device is named router1. You can view a list of available commands by pressing `?`.
 
 ```bash
Hello, this is FRRouting (version 8.4.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1#
```

You will now need to enter configuration by entering the command `configure`.

```bash
router2# configure
router2(config)# interface eth1
```

You can now configure the interface you are using for VRRP.

```bash
interface eth1
   vrrp 5 version 3
   vrrp 5 priority 200
   vrrp 5 advertisement-interval 1500
   vrrp 5 ip 10.10.20.2
   vrrp 5 ipv6 fd01::2
   end
write memory
```

Now you can check to see if it is working. On Server 1 you should see the following when running the command `show vrrp` on the VTY shell.

```bash
 Virtual Router ID                       5
 Protocol Version                        3
 Autoconfigured                          No
 Shutdown                                No
 Interface                               eth1
 VRRP interface (v4)                     vrrp4
 VRRP interface (v6)                     vrrp6
 Primary IP (v4)                         10.10.20.3
 Primary IP (v6)                         fe80::b868:88af:2e81:a44c
 Virtual MAC (v4)                        00:00:5e:00:01:05
 Virtual MAC (v6)                        00:00:5e:00:02:05
 Status (v4)                             Master
 Status (v6)                             Master
 Priority                                200
 Effective Priority (v4)                 200
 Effective Priority (v6)                 200
 Preempt Mode                            Yes
 Accept Mode                             Yes
 Advertisement Interval                  1500 ms
 Master Advertisement Interval (v4) Rx   1500 ms (stale)
 Master Advertisement Interval (v6) Rx   1500 ms (stale)
 Advertisements Tx (v4)                  5931
 Advertisements Tx (v6)                  5929
 Advertisements Rx (v4)                  0
 Advertisements Rx (v6)                  0
 Gratuitous ARP Tx (v4)                  1
 Neigh. Adverts Tx (v6)                  1
 State transitions (v4)                  2
 State transitions (v6)                  2
 Skew Time (v4)                          320 ms
 Skew Time (v6)                          320 ms
 Master Down Interval (v4)               4820 ms
 Master Down Interval (v6)               4820 ms
 IPv4 Addresses                          1
 ..................................      10.10.20.2
 IPv6 Addresses                          1
 ..................................      fd01::2
```

On server 2.

```bash
 Virtual Router ID                       5
 Protocol Version                        3
 Autoconfigured                          No
 Shutdown                                No
 Interface                               eth1
 VRRP interface (v4)                     vrrp4
 VRRP interface (v6)                     vrrp6
 Primary IP (v4)
 Primary IP (v6)                         ::
 Virtual MAC (v4)                        00:00:5e:00:01:05
 Virtual MAC (v6)                        00:00:5e:00:02:05
 Status (v4)                             Backup
 Status (v6)                             Backup
 Priority                                200
 Effective Priority (v4)                 200
 Effective Priority (v6)                 200
 Preempt Mode                            Yes
 Accept Mode                             Yes
 Advertisement Interval                  1500 ms
 Master Advertisement Interval (v4) Rx   1500 ms (stale)
 Master Advertisement Interval (v6) Rx   1500 ms (stale)
 Advertisements Tx (v4)                  0
 Advertisements Tx (v6)                  0
 Advertisements Rx (v4)                  5990
 Advertisements Rx (v6)                  5989
 Gratuitous ARP Tx (v4)                  0
 Neigh. Adverts Tx (v6)                  0
 State transitions (v4)                  1
 State transitions (v6)                  1
 Skew Time (v4)                          320 ms
 Skew Time (v6)                          320 ms
 Master Down Interval (v4)               4820 ms
 Master Down Interval (v6)               4820 ms
 IPv4 Addresses                          1
 ..................................      10.10.20.2
 IPv6 Addresses                          1
 ..................................      fd01::2
```

You can see Server 1 shows as the `Master` and Server 2 shows as the `Backup`.

That's it. VRRP is now configured using networkd and FRRouting on Debian 12.
