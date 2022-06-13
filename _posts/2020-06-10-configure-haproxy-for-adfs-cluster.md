---
title: Configure HAProxy for Microsoft ADFS Cluster
categories: [Guides,HAProxy]
tags: [guides,ubuntu,haproxy,linux,adfs,keepalived,vrrp,microsoft]
---

We had some aging hardware load balancers/reverse proxies that were no longer necessary for our setup. This lead to me working out how to replace them with some Ubuntu VMs.
In this guide I will include configuration of 2 HAProxy servers using keepalived for failover.

# Prerequisites
- 2 Ubuntu 20.04 Servers
- 3 available IP Addresses (Here we are using the 10.0.0.0/24 subnet)
    - 10.0.0.100 for keepalived
    - 10.0.0.101 for the MASTER server
    - 10.0.0.102 for the BACKUP server
- An ADFS Cluster

# Instructions
First you will need to install keepalived and haproxy onto each server

```bash
sudo apt install keepalived haproxy
```

## keepalived Configuration
Create a user for running keepalived scripts. This will add a user without a home directory and disable login.

```bash
sudo useradd -M keepalived_script
sudo usermod -L keepalived_script
```

Create the necessary configuration file on each server.

```bash
sudo vim /etc/keepalived/keepalived.conf
```

### MASTER Configuration File
Bits you will need to modify for your particular configuration:
- interface may be something other than ens160
- auth_pass can be changed to something random
- virtual_ipaddress

```
global_defs {
    router_id haproxy
    enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight 2
}

vrrp_instance haproxy {
    interface ens160
    state MASTER
    # MASTER requires a higher priority number than the SLAVE
    priority 101
    virtual_router_id 1
    authentication {
        auth_type AH
        auth_pass 12345678
    }
    virtual_ipaddress {
        10.0.0.100
    }
    track_script {
        chk_haproxy
    }
}
```

### BACKUP Configuration File
```
global_defs {
    router_id haproxy
    enable_script_security
}

vrrp_script chk_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight 2
}

vrrp_instance haproxy {
    interface ens160
    state BACKUP
    # BACKUP requires a lower priority number than the MASTER
    priority 100
    virtual_router_id 1
    authentication {
        auth_type AH
        auth_pass 12345678
    }
    virtual_ipaddress {
        10.0.0.100
    }
    track_script {
        chk_haproxy
    }
}
```

Enable, start, and check status of keepalived

```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```

The result of keepalived should look similar to this:
```
  ● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/lib/systemd/system/keepalived.service; enabled; vendor preset: enabled)
     Active: active (running) since
   Main PID: 3882 (keepalived)
      Tasks: 2 (limit: 19114)
     Memory: 5.1M
     CGroup: /system.slice/keepalived.service
             ├─3882 /usr/sbin/keepalived --dont-fork
             └─3893 /usr/sbin/keepalived --dont-fork
```

## HAProxy Configuration
This assumes you have the necessary certificate created and located under /etc/ssl/domain.com/certificate.pem

```bash
sudo vim /etc/haproxy/haproxy.cfg
```

Modify the following as needed.

```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    user haproxy
    group haproxy
    maxconn 40000
    ulimit-n 81000
    daemon
    ssl-default-bind-ciphers EECDH+AESGCM:EDH+AESGCM
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-server-ciphers EECDH+AESGCM:EDH+AESGCM
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    tune.ssl.default-dh-param 2048
    crt-base /etc/ssl/domain.com/

defaults
    log global
    mode http
    option httplog
    option dontlognull
    option forwardfor
    timeout client 30s
    timeout server 30s
    timeout connect 5s
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# Page to view statistics with username User and password Password
listen stats
    bind :9000
    mode http
    stats enable
    stats hide-version
    stats realm Haproxy\ Statistics
    stats uri /haproxy_stats
    stats auth Username:Password

# Frontend for HTTP with letsencrypt detection
frontend fe_http
    bind *:80
    # Test URI to see if it's a letsencrypt request
    acl letsencrypt path_beg /.well-known/acme-challenge/
    # Redirect HTTP to HTTPS with code 301 if not a letsencrypt request
    http-request redirect scheme https code 301 if !letsencrypt
    use_backend be_letsencrypt if letsencrypt

# Frontend for HTTPS
frontend fe_https
    bind *:443 ssl crt wildcard.pem
    acl sts ssl_fc_sni sts.domain.com
    use_backend be_sts if sts
    # This will be the webpage that is displayed if no matching FQDN is detected
    default_backend be_no-match

# Backends
backend be_letsencrypt
    server letsencrypt 10.0.0.103:80 check

backend be_sts
    balance roundrobin
    option httpchk GET /adfs/ls/IdpInitiatedSignon.aspx HTTP/1.1\r\nHost:\ sts.domain.com
    option forwardfor header X-Client
    http-check expect status 200
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    server adfs1 10.0.0.98:443 ssl verify none check check-sni sts.domain.com sni ssl_fc_sni inter 3s rise 2 fall 3
    server adfs2 10.0.0.99:443 ssl verify none check check-sni sts.domain.com sni ssl_fc_sni inter 3s rise 2 fall 3

backend be_no-match
    http-request deny deny_status 403
```

Make sure that you copy this configuration to your backup server.
Restart, and check status of haproxy.

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```
