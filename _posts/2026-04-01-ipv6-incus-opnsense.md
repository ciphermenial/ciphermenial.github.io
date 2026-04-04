---
title: Configuring IPv6 on OPNsense and Incus
categories: [Guides,IPv6]
tags: [guides,incus,lxc,linux,opnsense,radvd,ipv6,debian,network]
image: 
  path: /assets/img/title/ipv6-incus-opnsense.svg
---

I have previously written about configuring IPv6 on a Mikrotik router and an LXD server. I am writing a new and updated version, since I have switched to an OPNsense firewall and Incus servers. I am also writing this because I had to update all my changes due to switching from Aussie Broadband to Neptune for my ISP. Aussie Broadband was great but Neptune is cheaper with the same configuration but the benefit of a promise that the IPv6 Prefix Delegation will not change on my connection. I never had it change with Aussie Broadband but they couldn't promise it wouldn't.

I luckily purchased an Aoostar N1 Pro mini PC in October last year (2025) for AUD$220. This was a steal considering it comes with 12GB memory, 2x2.5gb Intel NICs, and a 256GB M.2 SSD. It's sad that you can't buy these anymore because they are a fatastic little device for OPNsense.

## IPv6 on OPNsense
There is nothing special about configuring this on OPNsense. The only bit that is different to the current [documentaion](https://docs.opnsense.org/manual/ipv6.html) on configuring 

### WAN Configuration

