---
title: Using Public IPv6 with LXD Containers
categories: [Guides,LXD]
tags: [guides,lxd,linux,mikrotik,routeros,ipv6,netplan]
image: 
  path: /assets/img/title/public-ipv6-lxd.svg
---
I recently gained access to a /48 IPv6 prefix for my connection. I previously had access to a /56 on my old ISP but their implementation of IPv6 is terrible and the /56 was not statically assigned to my connection.

With this proper IPv6 set up I wanted to configure it for complete support of IPv6 on my home lab. I am using a MikroTik hEX S as my router, which isn't great because IPv6 support on RouterOS is only really starting to pick up. The main reason I am using the hEX S is because I wanted POE-IN and an SFP port. The SFP port is because I annoyingly only have access to VDSL Internet access and I use a Proscend VDSL SFP in the hEX S.

I decided that I wanted to use IPv6 properly with my LXD host and have public IPs on the containers. I will go through the complete setup, including the RouterOS side.

## IPv6 on RouterOS

First we need to request the assigned prefix using IPv6 DHCP Client on RouterOS. To do this you simple add a client like so.

```bash
ipv6/dhcp-client/add interface=wan pool-name=ipv6-slaac-pool pool-prefix-length=64 request=address,prefix
```

Next is configuring an IP from that pool on the router. This again is simple with the following.

```bash
ipv6/address/add address=::1 from-pool=ipv6-slaac-pool interface=bridge
```

This will configure a /128 address from your provider and a dynamic IPv6 pool for use with SLAAC on your network. If you check in `ipv6/address/print` you will see the /128 address and the address added from the pool. It will look something like this.

```
#    ADDRESS                                      FROM-POOL        INTERFACE  ADVERTISE
0 DG 2001:db8:b00b:b00b:b00b:b00b:b00b:b00b/128                    sfp1       no       
1  G 2001:db8:b00b::1/64                          ipv6-slaac-pool  bridge     yes 
```

At this point, devices connected to your LAN (bridge) will be able to access IPs from this range.

## IPv6 on Ubuntu

I am using Ubuntu for my LXD host. To check if you have received an IPv6 address on your connection to the router you use the command `ip -6 address`. If you want you can shorten that by running `ip -6 a`. This will hopefully show an output like the following.

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP qlen 1000
    inet6 2001:db8:b00b:0:6f1b:64ff:fee0:623a/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591982sec preferred_lft 604782sec
    inet6 fe80::6f1b:64ff:fee0:623a/64 scope link 
       valid_lft forever preferred_lft forever
```

In this example my network adapter is named `eno1`. If you don't see your network adapter in this output, you will need to make some changes to your netplan configuration. I am not sure if this is required but I set it up this way.

I modified the config deployed on installation. I use vim when editing files.

```bash
sudo vim /etc/netplan/00-installer-config.yaml
```

You may need to add 2 lines to this configuration like this.

```diff
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:
      dhcp4: true
+     dhcp6: false
+     accept-ra: true
  version: 2
```

Then you apply this configuration by running `sudo netplan apply`

If you still don't see IPv6 on your adapter, you can disable and enable IPv6. I had to do this at one stage when playing around with making this work.

```bash
sudo sysctl -w net.ipv6.conf.eno1.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.eno1.disable_ipv6=0
```

You should now have a public IPv6 address on your host.

## Public IPv6 for LXD Containers

Now to configure LXD to use a public IPv6 range to use on your containers. This turned out to be extremely simple with a single command.

```bash
lxc network set lxdbr0 ipv6.address=2001:db8:b00b:1::1/64 ipv6.nat=false
```

I am statically assigning a /64 from my /48. You can assign any /64 besides 2001:db8:b00b::/64. In the example you can use 2001:db8:b00b:1::1/64 or for laughs you could use 2001:db8:b00b:b00b::1/64.

If you already had containers up and running and had a default IPv6 install on LXD, you might need to restart the containers to see the IPv6 address updated faster.

## Route to LXD /64

The last thing you need to do is create a route to the newly created IPv6 /64 subnet. You need the IPv6 address of the LXD host, which in my example is `2001:db8:b00b:0:6f1b:64ff:fee0:623a`. This is the IP you route to with the following command.

```bash
ipv6/route/add dst-address=2001:db8:b00b:1::1/64 gateway=2001:db8:b00b:0:6f1b:64ff:fee0:623a
```

You will now be able to add firewall rules in to forward traffic through to your container IPs. I would recommend locking down firewalls on the router and the LXD server as much as possible.
