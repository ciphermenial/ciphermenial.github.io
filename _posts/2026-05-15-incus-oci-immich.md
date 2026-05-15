---
title: Install Immich on App Containers with Incus
categories: [Guides,Incus]
tags: [guides,incus,linux,debian,oci,immich,nvidia,gpu,lxc]
image:
  path: assets/img/title/incus-oci-immich.svg
---

I originally install [Immich](https://immich.app/) on my Incus setup with Docker inside an LXC Container. Since I have been using OCI containers more I thought I would try get Immich running like that. In my setup I am installing Immich server on my main Incus host and the Immich Machine Learning on a secondary host that has an Nvidia GPU.

## Immich Server Container
### Prerequisites

- PostgreSQL DB
- Valkey or Redis
- [GitHub Container Repository added to Incus](https://blog.sifrmoja.xyz/posts/incus-oci/#add-an-oci-remote)

I already have a [PostgreSQL](https://www.postgresql.org/) LXC container and a [Valkey](https://valkey.io/) LXC container.

- A location to use for storage.

I have created a directory under my users home e.g. `/home/user/immich/` with the .env file and a data folder.

### Configuration

#### PostgreSQL
I use PGAdmin4 to manage my databases. I created a user named 'immich' and a database named 'immich' that is owned by that user. You will also need to add some extensions to the database with an admin account.

Immich requires the pgvector extension for its database. The best way to add it is through [official PostgreSQL repositories](https://www.postgresql.org/download/).

The extensions that you will need to add to the database manually are:
- cube
- earthdistance
- vector

#### Redis/Valkey
I bind my Valkey server to all available interfaces. I then use ACLs to allow connections to the Valkey.
This means you need to set the following in the `/etc/valkey/valkey.conf`

```bash
bind * -::*
aclfile /etc/valkey/users.acl
```

The ACL file looks like this. Make sure to change the password.

```bash
user default on +@all ~* >ThisIsAPassword
user immich on >ThisIsAPassword +@all ~* &*
```

#### Environment File

This is the bits necessary to include in the .env file.

```bash
# You can find documentation for all the supported env variables at https://docs.immich.app/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=./library

# The location where your database files are stored. Network shares are not supported for the database
DB_DATA_LOCATION=./postgres

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_z
ones#List
TZ=Australia/Adelaide

DB_USERNAME=immich
DB_PASSWORD=ThisIsAPassword
DB_DATABASE_NAME=immich
DB_HOSTNAME=postgresql.incus

REDIS_USERNAME=immich
REDIS_PASSWORD=ThisIsAPassword
REDIS_HOSTNAME=valkey.incus
```

### Launch OCI Image

```
incus launch ghcr:immich-app/immich-server:v2.7.5 --environment-file=/home/user/immich/.env
```

As of writing v2.7.5 is the current release. You will need to change that in the command as necessary.

### Data Directory

Stop the container and add the data directory.

```
incus config device add immich data disk path=/data source=/home/user/immich-app/data shift=true
```

This is mounting /data in the container to the data folder under my user folder. You can set this to any directory you choose.

> If you are using a directory on an NFS mounted share `shift=true` will stop the container from starting. [Here](https://discuss.linuxcontainers.org/t/add-a-mounted-to-host-nfs-target-as-disk-to-the-container-shift-true-got-error/24668) is a discussion about that.
{: .prompt-tip }

Start immich again and you can now access the web UI.

## Immich Machine Learning Container
This install is done on a machine with an Nvidia Quadro card.

### GPU Profile
To make things a bit easier I have created a profile with the parts for gpu passthrough. For more information about these commands, [I have an post that explains it](https://blog.sifrmoja.xyz/posts/jellyfin-gpu-passthrough/).

```bash
incus profile create nvidia-gpu
incus profile device add nvidia-gpu nvidia-gpu gpu pci=0000:01:00.0
incus profile set nvidia-gpu nvidia.runtime=true
incus profile set nvidia-gpu nvidia.require.cuda=true
incus profile set nvidia-gpu nvidia.driver.capabilities=all
```

> Make sure you have nvidia-container-tools installed. The instance will fail to start otherwise.
{: .prompt-tip }

### Launch OCI Image

```bash
incus launch ghcr:immich-app/immich-machine-learning:v2.7.5-cuda immich-ml --profile default --profile nvidia-gpu
```

It takes a decent amount of time for this container to build. If everything is configured correctly, you should be able to point your Immich server at this server and it will start working.
