---
title: Installing PostgreSQL and PGAdmin with Incus
categories: [Guides,PKI]
tags: [guides,incus,linux,debian,postgresql,lxc,postgresql,pgadmin4]
image:
  path: assets/img/title/postgresql-pgadmin-incus.svg
---
With my PostgreSQL setup, I have 2 LXC instances. One is PostgreSQL, and the other is for PGAdmin4 Web. I do this because I should learn the basics of SQL commands but I haven't.

## Launch Instances

```bash
incus launch images:debian/trixie postgresql
incus launch images:debian/trixie pgadmin
```

## Install PostgreSQL

Go into the shell of the instance named **postgresql**.

```bash
incus exec postgresql bash
```
