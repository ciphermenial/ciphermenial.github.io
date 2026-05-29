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

Install the base common package and run the script to add the PostgreSQL repositories.

```bash
apt install postgresql-common
/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

This will output the following.

```
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be trixie-pgdg.

Press Enter to continue, or Ctrl-C to abort.

Using keyring /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg
Writing /etc/apt/sources.list.d/pgdg.sources ...

Running apt-get update ...
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease
Hit:3 http://deb.debian.org/debian-security trixie-security InRelease
Hit:4 https://apt.postgresql.org/pub/repos/apt trixie-pgdg InRelease
Reading package lists... Done                           

You can now start installing packages from apt.postgresql.org.

Have a look at https://wiki.postgresql.org/wiki/Apt for more information;
most notably the FAQ at https://wiki.postgresql.org/wiki/Apt/FAQ
```

You can now install PostgreSQL latest version. As of this post it is version 18.

```bash
apt install postgresql
```

The reason I add the repositories is for extra extensions I need for other services like Immich.
