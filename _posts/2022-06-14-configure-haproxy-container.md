---
title: Setup & Configure HAProxy Container with Cloudflare Origins
categories: [Guides,HAProxy]
tags: [guides,debian,linux,lxc,incus,haproxy,cloudflare,certificates]
image: 
  path: /assets/img/title/cloudflare-haproxy.svg
---

This is the second guide in the series on how I setup my homelab. In my setup I use Cloudflare Origin Server between the world and my home server. My instructions will include all of the necessary configuration besides the required port forwards on your router. In my setup I only foward connections on port 443 from [Cloudflares](https://www.cloudflare.com/ips-v4) IPv4 ranges.

# Requirements

- Cloudflare account
- Public domain name
- Port 443 forwarded on your router to your LXD server.

# Create HAProxy Incus Container

To create the containers run the following.

```bash
incus launch images:debian/12 haproxy
```

This will create a new container with the name **haproxy** using a Ubuntu 22.04 image.

# Incus configuration

For this to work port 443 needs to be forwarded to the haproxy container. To do this you need haproxy to have a static IP.

```bash
incus config device override haproxy eth0 ipv4.address 10.10.10.254
incus restart haproxy
```

You need to forward HTTPS traffic from the host to the container. This creates a rule in nftables to pass the traffic through. You need to use your Incus hosts IP address for the listen address. Make sure your server has a static IP, in my example the server's IP is 10.0.2.15.

```bash
incus config device add haproxy https proxy listen=tcp:10.0.2.15:443 connect=tcp:10.10.10.254:443 nat=true
```

At this point all the configuration requirements for the container are complete. IF you want to view the configuration you can do the following.

`incus config show haproxy`

And the output will look similar to this.

```yaml
architecture: x86_64
config:
  image.architecture: amd64
  image.description: Debian bookworm amd64 (20240416_05:24)
  image.os: Debian
  image.release: bookworm
  image.serial: "20240416_05:24"
  image.type: squashfs
  image.variant: default
  volatile.base_image: e763cf30d37fa3a77d7c4ed0f74c084ad9480e0ec70ff515426e67a8225317c4
  volatile.cloud-init.instance-id: b4a2b8ba-0ccb-4cde-84a3-e11be74d00d5
  volatile.eth0.host_name: veth042b3414
  volatile.eth0.hwaddr: 00:16:3e:58:ab:47
  volatile.idmap.base: "0"
  volatile.idmap.current: '[{"Isuid":true,"Isgid":false,"Hostid":1000000,"Nsid":0,"Maprange":1000000000},{"Isuid":false,"I
sgid":true,"Hostid":1000000,"Nsid":0,"Maprange":1000000000}]'
  volatile.idmap.next: '[{"Isuid":true,"Isgid":false,"Hostid":1000000,"Nsid":0,"Maprange":1000000000},{"Isuid":false,"Isgi
d":true,"Hostid":1000000,"Nsid":0,"Maprange":1000000000}]'
  volatile.last_state.idmap: '[]'
  volatile.last_state.power: RUNNING
  volatile.uuid: 2ca6366b-2884-4c46-9791-d28619678059
  volatile.uuid.generation: 2ca6366b-2884-4c46-9791-d28619678059
devices:
  eth0:
    ipv4.address: 10.10.10.254
    name: eth0
    network: lxdbr0
    type: nic
  https:
    connect: tcp:10.10.10.254:443
    listen: tcp:10.0.2.15:443
    nat: "true"
    type: proxy
ephemeral: false
profiles:
- default
stateful: false
description: ""
```

# Install and Configure HAProxy

To change to the terminal of the HAProxy container you can do the following.

`incus exec haproxy bash`

Now I update apt repositories and install all updates. Then install haproxy.

```bash
apt update
apt full-upgrade
apt install haproxy
```

We need to configure the haproxy.cfg. For a write-up of my complete finalised HAProxy configuration click [here](https://ciphermenial.github.io/posts/my-haproxy-config/).

```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    user haproxy
    group haproxy
    daemon
    maxconn 40000
    ulimit-n 81000
    crt-base /etc/haproxy/certificates/

defaults
    mode http
    option httplog
    option dontlognull
    option forwardfor
    log global
    timeout client 30s
    timeout server 30s
    timeout connect 5s

# HTTP Frontend
frontend fe_http
    bind *:80
    http-request redirect scheme https code 301

# HTTPS Frontend
frontend fe_https
    bind *:443 ssl crt domain.com.pem

    acl service ssl_fc_sni service.domain.com

    use_backend be_service if service

    # This redirects to a failure page
    default_backend be_no-match

backend be_no-match
    http-request deny deny_status 403

backend be_service
    server service service.lxd:80 check
```

When we add other containers with services I will show how to configure this file for access to the services.

# Configure Cloudflare

## Edge Certificate

When you sign up to Cloudflare with your domain you receive a Universal SSL certificate which is enabled by default. I am not sure what the default settings are but mine is set as follows but you can decide how you want to manage it. If you need backwards compatibility for TLS, you can set those setting as you please.

- Always Use HTTPS: True
  
- Minimum TLS Version: TLS 1.3
  
- Opportunistic Encryption: True
  
- TLS 1.3: True
  
- Automatic HTTPS Rewrites: True
  

## Origin Certificate

This is the certificate you will install into your haproxy. For an explanation please read through the Cloudflare documentation for this [here](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/).

[Here are the instuctions](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/) to create an Origin CA certificate from Cloudflare docs

After the Origin Certificate is created you will be taken to a page that shows the Origin Certificate and the Private Key.

We will paste both sections of those into a single PEM file on the HAProxy server.

You will need to make a directory called certificates under `/etc/haproxy` and create a file using your editor of choice

```bash
mkdir /etc/haproxy/certificates
vim /etc/haproxy/certificate/domain.com.pem
```

Paste in the Origin Certificate and Private Key. It will look something like this.

```
-----BEGIN CERTIFICATE-----
MIIEpAIBAAKCAQEArxHLfi123Z1mIM7H4xKsDwbxfVvsPVxSuwR3kxLD1nJE/YCp
pTF/IJHNVFd1eR/PRv2P5i8i8c/u5++p4NC7nonZv0w1dDQTcIXx9Q9BfLytO4Al
u/sz6qZ3F9gEEQFnaaZPHkpYoWq95JMIKxx6rrnpW6VUz/4+b3Uxu1wdJ9LeGXtf
ypvmbXmuvakmGz7CGxhWYNZ6JsoqkBTLR61LjUHrg1G2HexrNv1iWFx4GhWn8wXE
b8uDUp/xbsLEQM3V4z7LCAqLhh/elKG/TzKekU1QrJC1pwyKRUJ6+MryITEUkH5K
9m/y4W7e27wP5AwPBctoKJDVerfuFopQMJ8qgQIDAQABAoIBAQCAqS9IF9mXnSmF
SvKT6xEQKiYn3vqLTeJvFyVZrRzH6UrSk1AZ23p2UTD5nxzyW3JV1dt/a3zfAdWu
FvBeDIkWRnEEqdlPAUaYF5huZTvXlEIrzE3vDPpmpNg5acPzS3jYqCTVOgZQ+sV7
yqLiLBfteSwK8kKWaV8xQou+CkBTFuoA3SJqTH3P+G0IEWEPJ3llM1lJQq1J4TrV
1zz7DrB/xvIB2+ZyD5mS338geVJs2QLu7m/UvERNf6wqQzlxGLNv0r1pmbwjcKTw
vUI71CU5ETtY0XZQ/B9DfWoc0InHRo+9KzvqX03SUz1LjMNOoBF+2G57jSgk/81/
UmhstL8BAoGBAN/63GnQddLPkAuG+uQ9UA3CSd/CsAYxFyZFW0pIylLzyK5HWZRm
dlQ6fJHFA3P3KR79llxQmCseKCAOJPoEkmEiMKEPpqNXq3yFMc5IEuCuhe/KIKv6
NQYiS8DuGn7y/LYIBN8mM3WXhFAyPCNM/UVAVO/LhGhEGKlKhCPfslRJAoGBAMgY
7Xr8r0ABuD5fGxBDTPNA8rMZ6K1W48j1TzTppiAWuSkeC5rZgpJeySIL9Q3iMCqa
pQ3VBYBALszA6McsCp4PlKx85DS8JymFUO14tQzMkdIQyBqrs7ur6+rzZQu0p8cN
VnvrpT1kRhubDYFVR88HTAFo9NmhphSEwj8dgrR5AoGAThwHF+O54z29Zze4cTYs
n8+8wYr8pfwirZcMYhiGbm1T8+swAz/ETlVjMda6AIwWTBd1g1Yb6xWGOr+UB5jm
j3dD7DcwDtC5HiC5IM4jvzU9wkUEJdWI/k2hi3O9y73jgXvEbym8Umr3mpwaOtlT
jf4EYOfhkhcFXqx87qHJZ/kCgYA7EGihIg9U9Gz/NDGX5lXDhAtf5Kjy6bAJNKfx
tXpNBIgZY/4G8meBbystupvWQkr3eHh6EcQy7D8kP1k22YA00eKP27m8+0EQF4Mg
5b2DjqsId92pSb+fCQt1ae0MvIG91ukNYSyAZ6XuJiGhaJvut3eu/t0vlHCio+F2
oe5f+QKBgQDIPi0Rah6OMvqMj5S4aPlC6/TCYIbDyK0wplFdO+a4sLiI/Nlds2ck
p5NXuveBBTBdCbQ6iv02Ultt2GXo7Kv/GIGAUI6z0238bnoVvtoLeN7slkC1Wptc
Ec8pgt0KyGKzCFH2gtIeUl/04RKEIjyAv/7D3gd1d0W6Fo9xE7v4yg==
-----END CERTIFICATE-----
-----BEGIN PRIVATE KEY-----
MIIEogIBAAKCAQEAiocVnTHMYU2ItRCsQOpl4b5zsRBHyncJ5nDzTJg771xRMzwV
8c1uN2JwvO5smo/Tc1Ri4Od0WFN5HZnsYMhWayyHKVWyCEGLpMWQBSH9wKR+hYMD
6NiMAfYpEqWAB4TALRK2fhwYVXFXM1RLQlP4YP0q7ZfmEYFeq9aAgDJN0I3HThTQ
4iLEdzoI2Ak6HFAQxpGFFcVq/hhygu4WWX/LyM4rrajkM6MEb+gxWjNfZUu6M3NS
1W6v5ryZn43SzF0GJLTiPPFapQ5XIxsbMr781hVHzOaa/p8u+eGTb5n3ZphbkM2Q
uscU/YD+ElUobc74QGma+MXxssRzQvHUwEOIzwIDAQABAoIBABi1o8tYWKZ6mAoE
IVWq+eVcfXJ1/vhEZ4WtXBirhvVZODq1Wwy4ohJLAuUQelrPkN4fjUukvYIL0azQ
CfPxiEixtqJO4OTMHEaV3uyrdYHpVZAnIIlmJwMqj4T99Gpi6Yygq+CuzkBfaTiE
rq/0HnfecMvUrnss4mAwcNdtIagzfnHaf3DcSZwIyoRniMQQKxvdhRuSM+n1qar7
GQIqWHMS0qIofYOvTjIByWfp/IyY8ltTzmEJgyaFcrT0tC+WJ2XC4hbYddRRTWAf
OoZhtbsQUY8lfUiGXB4G+FFSVcvOgR4la0+MKYHvr15bKAp2nJ6hhwM/J9mTm0I7
JFb5TMECgYEA0Lon/8sTVvubrHT44R91ScSYp9qyrPzICLgIttzPyzoq3LrNGwlu
v790AWe0n8uUfkM42TcodJCbn3VCL6O6mtQgfHXt+3A7xaI6pON9r8E927XwsCpq
XJpfzWYktMDafTB6ltmd4EWKM1/igXGjKkdZRJ8rV/Zefu8Ff5Z6918CgYEAqebQ
c3a4pz7coTYgOH+ST0NOHijTEi56r8SSgddGsrJko8Oem2RZoXf/nAwskrHQ1cuQ
2fSliEn7zzv1gVcYyLG3vvTQYEjFCp4qHc1YP7S/UdKRnDYD7dN15ZCxPUHU4t+L
wBaxnSzKkicdUyntBKZIO0QNMjmVzrVz60z2FJECgYAxOoaulNXl4QfxX9FHP2Up
Vd3vUOxtUl1XeRhNEL1NoFV1o/U2GD5vqRcSMcRvH9PRB8fDq3e2LlkV/dDzbXlY
hQl4cVQExo7CaSXNt/3v0vLk+/9dfVOCrcJErn+fxhCCEEoJhB/xQlV7EnVYtFWY
ZiWOwr+1Sl01MOiqE/LCnwKBgB7oQC9g/4JdKyGgiQf+HQ2SPtm5r3v1PJhQ+B3q
nY/QaAJqiaXXAX8gJz2p8UnWUxkxaO5dVOeQHeC7FZQr1fRccAKq4mVBl6aw0xSM
0Gr2ZH9sANUb9mcDOsVCJxvvp9yFshSFjFX9WfRwbSM900IvRaCSZpwmYZwy4h2B
6JohAoGAEjNEnQmjno4cXLBrcOnHblZhM3YyD81fCZ620wzTSPneOeOaFPOQaDLk
V+8t/qJ2h2Z+SszWtB8GgtkRSmotWv+QPXkxfMi/WMMKcEOHATg+D64wuWTLToJf
9ABFk+hNCdW5cLNnBqOwgYzKSniYsx/j+Y/Z9hXbgoPXwyWT7vY=
-----END PRIVATE KEY-----
```

You are now ready to create your first web service container.
