---
title: Aruba CX & Azure ExpressRoute
categories: [Guides,Networking]
tags: [guides,azure,expressroute,aruba,aos-cx,bgp]
image:
  path: /assets/img/title/cx-expressroute.svg
---

I recently had to configure BGP on 2 pairs of Aruba CX switches configured as VSX pairs.

## Requirements

To follow this setup completely you will need:

- ExpressRoute Circuit configured with ISP
- Q-in-Q configured on the circuit by your ISP
- Aruba CX in a square topology (it shouldn't take much to modify for different configurations)

## ExpressRoute Setup

Once the circuit is configured by your ISP, you will need to create a peering. In this guide I am using Azure private peering and will only be configuring IPv4.

Microsoft only allow for a single VLAN and /30 for primary and secondary connections.

The configuration I am using is as follows:

![](/assets/img/2022-12-21-aruba-cx-expressroute/expressroute-peering.png)

## Aruba CX Configuration

The following configuration is what I ended up with after lots of testing. I am not completely sure whether it is the best option but it is working well.

I used the following subnets and VLANs for this:

- 10.1.0.0/24 for iBGP
  
- 10.2.0.0/30 for Azure ExpressRoute primary BGP
  
- 10.2.0.4/30 for Azure ExpressRoute secondary BGP
  
- VLAN 100 for iBGP
  
- VLAN 200 for Azure ExpressRoute
  
- 10.0.0.0/8 for advertised routes (you would want to change this to the smallest amount required)

In my configuration it is a bit mixed up with primary and secondary data centres. The VSX pairs primary and secondary is opposite to the ISP Router primary and secondary configuration. The primary ExpressRoute hand-off is connected to the secondary data centre primary switch. Here is a diagram of my configuration:

![](/assets/img/2022-12-21-aruba-cx-expressroute/aruba-cx-layout.png)

### Configuration

#### Core Switch 1

```
vlan 100
    name iBGP
vlan 200
    name Azure ExpressRoute
!
interface 1/1/41
    description Secondary Azure ExpressRoute Link
    no shutdown
    no routing
    speed 1000-full
    vlan trunk native 1
    vlan trunk native 200
!
interface vlan 100
    description iBGP interface
    ip address 10.1.0.1/24
interface vlan 200
    description Azure ExpressRoute eBGP interface
    ip address 10.2.0.5/30
!
ip prefix-list ALL_ROUTES seq 10 permit 0.0.0.0/0 le 32
ip prefix-list AZURE_SUMMARY seq 10 permit 10.20.0.0/16 le 24
ip prefix-list DEFAULT_ROUTE seq 10 permit 0.0.0.0/0
ip prefix-list LOCAL_SUMMARY seq 10 permit 10.0.0.0/8
!
route-map AZURE-PEER-IN permit seq 10
    match ip address prefix-list AZURE_SUMMARY
    set local-preference 90
route-map AZURE-PEER-IN deny seq 100
    match ip address prefix-list ALL_ROUTES
route-map AZURE-PEER-OUT permit seq 10
    match ip address prefix-list LOCAL_SUMMARY
    set as-path prepend 65500
route-map AZURE-PEER-OUT deny seq 100
    match ip address prefix-list ALL_ROUTES
!
bfd
!
router bgp 65500
    bgp router-id 10.1.0.1
    neighbor 10.2.0.6 remote-as 12076
    neighbor 10.2.0.6 fall-over bfd
    neighbor 10.1.0.2 remote-as 65500
    neighbor 10.1.0.3 remote-as 65500
    neighbor 10.1.0.4 remote-as 65500
    address-family ipv4 unicast
        neighbor 10.2.0.6 activate
        neighbor 10.2.0.6 route-map AZURE-PEER-IN in
        neighbor 10.2.0.6 route-map AZURE-PEER-OUT out
        neighbor 10.1.0.2 activate
        neighbor 10.1.0.2 next-hop-self
        neighbor 10.1.0.3 activate
        neighbor 10.1.0.3 next-hop-self
        neighbor 10.1.0.4 activate
        neighbor 10.1.0.4 next-hop-self		
        network 10.0.0.0/8
    exit-address-family
```

#### Core Switch 2

```
vlan 100
    name iBGP
!
interface vlan 100
    description iBGP interface
    ip address 10.1.0.2/24
!
router bgp 65500
    bgp router-id 10.1.0.2
    neighbor 10.1.0.1 remote-as 65500
    neighbor 10.1.0.3 remote-as 65500
    neighbor 10.1.0.4 remote-as 65500
    address-family ipv4 unicast
        neighbor 10.1.0.1 activate
        neighbor 10.1.0.3 activate
        neighbor 10.1.0.4 activate
    exit-address-family
```

#### Core Switch 3

```
vlan 100
    name iBGP
vlan 200
    name Azure ExpressRoute
!
interface 1/1/41
    description Primary Azure ExpressRoute link
    no shutdown
    no routing
    speed 1000-full
    vlan trunk native 1
    vlan trunk allowed 200
!
interface vlan 100
    description iBGP interface
    ip address 10.1.0.3/24
interface vlan 200
    description Azure ExpressRoute eBGP interface
    ip address 10.2.0.1/30
!
ip prefix-list ALL_ROUTES seq 10 permit 0.0.0.0/0 le 32
ip prefix-list AZURE_SUMMARY seq 10 permit 10.20.0.0/16 le 24
ip prefix-list DEFAULT_ROUTE seq 10 permit 0.0.0.0/0
ip prefix-list LOCAL_SUMMARY seq 10 permit 10.0.0.0/8
!
route-map AZURE-PEER-IN permit seq 10
    match ip address prefix-list AZURE_SUMMARY
    set local-preference 100
route-map AZURE-PEER-IN deny seq 100
    match ip address prefix-list ALL_ROUTES
route-map AZURE-PEER-OUT permit seq 10
    match ip address prefix-list LOCAL_SUMMARY
route-map AZURE-PEER-OUT deny seq 100
    match ip address prefix-list ALL_ROUTES
!
bfd
!
router bgp 65500
    bgp router-id 10.1.0.3
    neighbor 10.2.0.2 remote-as 12076
    neighbor 10.2.0.2 fall-over bfd
    neighbor 10.1.0.1 remote-as 65500
    neighbor 10.1.0.2 remote-as 65500
    neighbor 10.1.0.4 remote-as 65500
    address-family ipv4 unicast
        neighbor 10.2.0.2 activate
        neighbor 10.2.0.2 route-map AZURE-PEER-IN in
        neighbor 10.2.0.2 route-map AZURE-PEER-OUT out
        neighbor 10.1.0.1 activate
        neighbor 10.1.0.1 next-hop-self
        neighbor 10.1.0.2 activate
        neighbor 10.1.0.2 next-hop-self
        neighbor 10.1.0.4 activate
        neighbor 10.1.0.4 next-hop-self		
        network 10.0.0.0/8
    exit-address-family
```

#### Core Switch 4

```
vlan 100
    name iBGP
!
interface vlan 100
    description iBGP interface
    ip address 10.1.0.4/24
!
router bgp 65500
    bgp router-id 10.1.0.4
    neighbor 10.1.0.1 remote-as 65500
    neighbor 10.1.0.2 remote-as 65500
    neighbor 10.1.0.3 remote-as 65500
    address-family ipv4 unicast
        neighbor 10.1.0.1 activate
        neighbor 10.1.0.2 activate
        neighbor 10.1.0.3 activate
    exit-address-family
```
