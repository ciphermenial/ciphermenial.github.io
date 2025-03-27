---
title: Home Lab
icon: fas fa-network-wired
order: 5
---

I am writing a bunch of guides on how I configure my homelab. My goal is to use [Incus](https://linuxcontainers.org/incus/) as the basis of my setup. I started with LXD a long time ago and when Canonical ruined that and Incus was created, I switched to that. It is incredible seeing how rapidly Incus is being developed. The speed at which, the addition where you can easily nest Docker in [LXC](https://linuxcontainers.org/lxc/introduction/) to now having basic support for the [Open Container Initiative](https://opencontainers.org/) (App Containers), is incredible.

I try to use IPv6 as much as possible as well. If you don't know IPv6 well and want to learn more, I highly recommend setting up Incus with IPv6 and playing around. Also make sure that you have an Internet Service Provider that supports IPv6 standards. I use [Aussie Broadband](https://www.aussiebroadband.com.au/) who give out a /48 to all customers (this is how all ISPs should behave).  They also enable you to set your reverse DNS for your range.

I use my homelab to learn new things that I often end up implementing in my work as a sysadmin. A lot of my guides will be looking at using IPv6 only or default if I need IPv4.

# Todo list
- [x] [Configure Incus on Debian](/posts/configure-incus-on-debian/)
- [x] [Setup & Configure HAProxy Container with Cloudflare Origins](/posts/configure-haproxy-container/)
- [x] [Configure Incus for Docker](/posts/configure-incus-for-docker/)
- [x] [Setup SMTP Outbound Server with DKIM](/posts/smtp-outbound-server/)
- [ ] Install & Configure MariaDB Container
- [ ] Install & Configure PostgreSQL Container
- [ ] Install & Configure Keycloak Container
- [ ] Install & Configure Apache Guacamole Container
