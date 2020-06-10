---
layout: post
title: Configure HAProxy for Microsoft ADFS Cluster
categories: [Network,Microsoft]
excerpt: How to configure HAProxy to work with a Microsoft Active Director Federation Server Cluster
---
# Introduction
We had some aging hardware load balancers/reverse proxies that were no longer necessary for our setup. This lead to me working out how to replace them with some Ubuntu VMs.
In this guide I will include configuration of 2 HAProxy servers using keepalived for failover.

# Prerequisites
- 2 Ubuntu 20.04 Servers
- 3 available IP Addresses (Here we are using the 10.0.0.0/24 subnet)
    - 10.0.0.100 for keepalived
    - 10.0.0.101 for the MASTER server
    - 10.0.0.102 for the BACKUP server

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
    virtual_router_id 51
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
    virtual_router_id 51
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

