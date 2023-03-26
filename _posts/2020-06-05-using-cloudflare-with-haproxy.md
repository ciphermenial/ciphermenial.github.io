---
title: Using Cloudflare With HAProxy
categories: [Guides,HAProxy]
tags: [guides,haproxy,linux,cloudflare,openwrt]
---

A while ago I switched to using [Cloudflare](https://www.cloudflare.com) for my domain names [DNS](https://en.wikipedia.org/wiki/Domain_Name_System). The main reason I did this was for dynamic DNS since I had a dynamic IP on my home Internet connection. I then looked into what else I could use Cloudflare for and over time have taken advantage of more of their free options.

I was looking at setting up [HAProxy](https://www.haproxy.org) anyway because I have a server that I use to play with all kinds of web services. I have a [Icinga2](https://icinga.com) instance for monitoring, a [Bookstack](https://www.bookstackapp.com) setup for taking notes, a [Home Assistant](https://www.home-assistant.io) install, and more. To make these easily accessible externally I wanted to use HAProxy with [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication). While looking into this I discovered Cloudflare Origin CA and use it in the following instructions.

# Requirements
To follow this setup completely you will need:
- An [OpenWrt](https://openwrt.org) based router
- A public domain name
- A Cloudflare account
  - [Getting Started](https://developers.cloudflare.com/fundamentals/get-started/)

# Cloudflare Setup
After you have followed through the [Cloudflare Getting Started guide.](https://developers.cloudflare.com/fundamentals/get-started/)

You will then need to configure [Cloudflare's Universal SSL](https://blog.cloudflare.com/universal-ssl-encryption-all-the-way-to-the-origin-for-free) by following the guide [https://support.cloudflare.com/hc/en-us/articles/115000479507](https://support.cloudflare.com/hc/en-us/articles/115000479507) to create a Cloudflare Origin Certificate to install onto your router.

Once you have the certificate and key, you can combine them together to create a .pem file. You will then copy this .pem to somewhere on your router. I've placed mine under ```/etc/ssl/cloudflare/domain.com.pem```

Also remember to backup this file somewhere secure as in that location it won't be saved during an upgrade of OpenWrt.

# HAProxy Configuration
This is the configuration I have used but I am not sure if it is the best way to go about it. I would love some feedback about whether there is a better way to manage it. You will usually need to install the haproxy package from opkg.

```bash
opkg install haproxy
```

The haproxy configuration file is located at `/etc/haproxy.cfg`

> This is a poor configuration that I used when still learning. I have changed a lot of what I do since then. You can see that [here](https://ciphermenial.github.io/posts/my-haproxy-config/).
{: .prompt-warning }

```
# Global parameters
defaults
    timeout http-request 5s
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-keep-alive 4s
    option http-server-close

global
    log 10.0.0.10 local0
    maxconn 32000
    ulimit-n 65535
    uid 0
    gid 0
    daemon
    nosplice
    tune.ssl.default-dh-param 2048
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDH
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-server-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:EC
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets

# Frontend for SNI Passthrough
frontend frontend_snipt
    bind *:443
    mode tcp
    log global
    option forwardfor header X-Forwarded-For
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    # This will decide which backend to pass the traffic to
    # based on the FQDN
    use_backend backend_snipt_1 if { req_ssl_sni -i 1.domain.com }
    use_backend backend_snipt_2 if { req_ssl_sni -i 2.domain.com }
    use_backend backend_snipt_3 if { req_ssl_sni -i 3.domain.com }
    use_backend backend_snipt_4 if { req_ssl_sni -i 4.domain.com }
    use_backend backend_snipt_5 if { req_ssl_sni -i 5.domain.com }
    default_backend backend_1

# Backend for SNI Passthrough
backend backend_snipt_1
    mode tcp
    server localhost 127.0.0.1:7000 check

backend backend_snipt_2
    mode tcp
    server localhost 127.0.0.1:7001 check

backend backend_snipt_3
    mode tcp
    server localhost 127.0.0.1:7002 check

backend backend_snipt_4
    mode tcp
    server localhost 127.0.0.1:7003 check

backend backend_snipt_5
    mode tcp
    server localhost 127.0.0.1:7004 check

# Normal frontend
frontend frontend_1
    bind *:7000 ssl strict-sni crt /etc/ssl/cloudflare/domain.com.pem
    mode http
    use_backend backend_1

frontend frontend_2
    bind *:7001 ssl strict-sni crt /etc/ssl/cloudflare/domain.com.pem
    mode http
    use_backend backend_2

frontend frontend_3
    bind *:7002 ssl strict-sni crt /etc/ssl/cloudflare/domain.com.pem
    mode http
    use_backend backend_3

frontend frontend_4
    bind *:7003 ssl strict-sni crt /etc/ssl/cloudflare/domain.com.pem
    mode http
    use_backend backend_4

frontend frontend_5
    bind *:7004 ssl strict-sni crt /etc/ssl/cloudflare/domain.com.pem
    mode tcp
    option clitcpka
    timeout client 3h
    timeout server 3h
    use_backend backend_5

# Normal backend
backend backend_1
    mode http
    server server01 10.0.0.10:80 check

backend backend_2
    mode http
    server server01 10.0.0.10:8080 check

backend backend_3
    mode http
    server server02 10.0.0.254:80 check

backend backend_4
    mode http
    server server01 10.0.0.10:8081 check

backend backend_5
    mode http
    server server01 10.0.0.10:8082 check
```

With this you can also make the LuCI interface available externally and know that the traffic will all be encrypted. This setup offloads all the TLS encryption.
You will need to also go to System > Startup in LuCI and start the haproxy service.

# Firewall Configuration
The last thing you need to make this all work is to open port 443 on the router. Browse to Network > Firewall > Traffic Rules and under the section **Open ports on router** add a rule for TCP for port 443.

## Extra Security
Thanks to Cloudflare having a nice text list of their IP ranges it is trivial to set your firewall to only allow connections to port 443 from CloudFlare.

In the standard OpenWrt install it will not have all the packages required for this step. Install the required packages from command line or in LuCI

opkg install curl
opkg install ipset

## Scheduled Task
To configure a scheduled task in OpenWrt is really simple. I did it through the Web UI as follows

1. Open up the Web UI
2. Go to System > Scheduled Tasks
3. In the input window paste in the following\
```0 0 * * * curl https://www.cloudflare.com/ips-v4 > /etc/cfip.v4```
4. Click Submit
5. Go to System > Startup
6. Restart cron

This will create a task that will run at midnight and copy the contents of the text file to a local file name cfip.v4 under /etc For more information about crontab and how it works click [here](https://www.adminschoice.com/crontab-quick-reference).

To make sure that file is created straight away it is a good idea to run the command on the router.

```bash
curl https://www.cloudflare.com/ips-v4 > /etc/cfip.v4
```

## ipset Configuration
ipset can be configured like you would on any iptables firewall but I have used OpenWrt config method. It is as simple as modifying the /etc/config/firewall file and adding the following.

```
config ipset
    option name 'cf'
    option match 'src_net'
    option storage 'hash'
    option enabled '1'
    option loadfile '/etc/cfip.v4'
    config rule
    option target 'ACCEPT'
    option src 'wan'
    option proto 'tcp'
    option dest_port '443'
    option name 'Cloudflare Inbound'
    option ipset 'cf'
```

You will then need to restart the firewall service

```bash
/etc/init.d/firewall restart
```

To check whether the ipset has been created you can run the command `ipset list cf` which should display results that look like the following.

```
Name: cf
  Type: hash:net
  Revision: 6
  Header: family inet hashsize 1024 maxelem 65536
  Size in memory: 1096
  References: 1
  Number of entries: 14
  Members:
  198.41.128.0/17
  172.64.0.0/13
  197.234.240.0/22
  141.101.64.0/18
  162.158.0.0/15
  190.93.240.0/20
  103.31.4.0/22
  188.114.96.0/20
  104.16.0.0/12
  103.22.200.0/22
  131.0.72.0/22
  173.245.48.0/20
  108.162.192.0/18
  103.21.244.0/22
```
