---
title: Home Lab
icon: fas fa-network-wired
order: 5
---

I am writing a bunch of guides on how I configure my homelab. My goal is to use [Incus](https://linuxcontainers.org/incus/) as the basis of my setup. I started with LXD a long time ago and when Canonical ruined that and Incus was created, I switched to that. It is incredible seeing how rapidly Incus is being developed. The speed at which, the addition where you can easily nest Docker in [LXC](https://linuxcontainers.org/lxc/introduction/) to now having basic support for the [Open Container Initiative](https://opencontainers.org/) (App Containers), is incredible.

I try to use IPv6 as much as possible as well. If you don't know IPv6 well and want to learn more, I highly recommend setting up Incus with IPv6 and playing around. Also make sure that you have an Internet Service Provider that supports IPv6 standards. I use [Neptune Internet](https://www.neptune.net.au/), (previously [Aussie Broadband](https://www.aussiebroadband.com.au/)[^aussie]) who give out a /48 to all customers (this is how all ISPs should behave).  They also enable you to set your reverse DNS for your range.

I use my homelab to learn new things that I often end up implementing in my work as a sysadmin. A lot of my guides will be looking at using IPv6 only or default if I need IPv4.

# Todo list
- [x] [Configure Incus on Debian](/posts/configure-incus-on-debian/)
- [x] [Setup & Configure HAProxy Container with Cloudflare Origins](/posts/configure-haproxy-container/)
- [x] [Configure Incus for Docker](/posts/configure-incus-for-docker/)
- [x] [Setup SMTP Outbound Server with DKIM](/posts/smtp-outbound-server/)
- [x] [Install & Configure MariaDB Instance](/posts/install-mariadb-debian/)
- [x] [Install & Configure PostgreSQL Instance](/posts/postgresql-pgadmin-incus/)
- [ ] Install & Configure Keycloak Instance
- [ ] Install & Configure Apache Guacamole Instance

# Recommended Projects
- Audio
  - [Navidrome](https://navidrome.org/) as a music streaming server.
  - [beets](https://beets.io/) to tag songs.
  - [lrclib](https://lrclib.net/) for adding lyrics to songs using beets.
  - [Feishin](https://feishin.net/) Client for Navidrome (Desktop & Web).
  - [Symfonium](https://symfonium.app/) Client for Navidrome (Adnroid & Android TV)[^closedsource].
  - [slskd](https://github.com/slskd/slskd) Web client for SoulSeek.
- Visual
  - [Jellyfin](https://jellyfin.org/) as a series/movie streaming server.
  - [Immich](https://immich.app/) as a photo management server.
  - [Radarr](https://radarr.video/) to automate movie management.
  - [Sonarr](https://sonarr.tv/) to automate series management.
  - [Seerr](https://seerr.dev/) to allow users request and add media.
- Life Management
  - [Home Assistant](https://www.home-assistant.io/) for your samrt home.
  - [Readeck](https://readeck.org/en/) for bookmarks..
  - [Vaultwarden](https://vaultwarden.com/) for passwords.
  - [Firefly III](https://firefly-iii.org/) for personal finances.
  - [Joplin](https://joplinapp.org/) for note taking.
  - [Monica](https://www.monicahq.com/) for personal relationship managment.[^monica]
  - [Paperless-ngx](https://docs.paperless-ngx.com/) for documents.
  - [Tandoor Recipes](https://docs.tandoor.dev/) for recipes.
 
[^aussie]: Aussie Broadband is still a fantastic choice. My reason for switching was purely pricing.
[^closedsource]: Not OSS.
[^monica]: This has not been maintained properly for a long time but the maintainers have promised a new version recently.
