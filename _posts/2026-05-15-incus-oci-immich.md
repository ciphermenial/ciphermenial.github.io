---
title: Install Immich on App Containers with Incus
categories: [Guides,Incus]
tags: [guides,incus,linux,debian,oci,immich,nvidia,gpu,lxc]
image:
  path: assets/img/title/incus-oci-immich.svg
---

I originally install Immich on my Incus setup with Docker inside an LXC Container. Since I have been using OCI containers more I thought I would try get Immich running like that. In my setup I am installing Immich server on my main Incus host and the Immich Machine Learning on a secondary host that has an Nvidia GPU.

## Immich Server Container
### Prerequisites
I already have a PostgreSQL LXC container and a Redis (need to change to valkey) LXC container.
