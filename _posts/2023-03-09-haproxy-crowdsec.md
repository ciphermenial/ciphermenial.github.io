---
title: Setup & Configure CrowdSec HAProxy Bouncer
categories: [Guides,HAProxy]
tags: [guides,ubuntu,linux,haproxy,crowdsec]
image:
  path: /assets/img/title/crowdsec-haproxy.svg
---

With the recent release of the official HAProxy [bouncer](https://doc.crowdsec.net/docs/next/bouncers/haproxy) for [Crowdsec](https://www.crowdsec.net/) I thought I would give it a go. In doing this I found out that CrowdSec bouncer was detecting the IP using source IP, which required the use of CF-Connecting-IP to be set as the source IP. I was having issues because I have a loop in my HAProxy configuration that meant it was grabbing 127.0.0.1 as the client IP. I have resolved all of these issues and it is now working perfectly.

> This guide assumes you have HAProxy 2.5 or higher running. I am using Ubuntu.
{: .prompt-info }

## Requirements

- HAProxy 2.5 or higher
- Cloudflare account
- [CrowdSec account](https://app.crowdsec.net/signup)
- Backend Web Services

## Install CrowdSec Agent & Bouncer

There isn't much to this. You simply need to run the following.

```bash
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash
sudo apt install crowdsec
```

If you aren't installing it on Ubuntu you can go [here](https://www.crowdsec.net/product/agent) for instructions.
The bouncer is also installed from apt.

```bash
sudo apt install crowdsec-haproxy-bouncer
```

## CrowdSec Configuration

It is simple to add your CrowdSec agent to the console. Sign into your CrowdSec console and under the Instances section, click [Instances](https://app.crowdsec.net/instances).

Click on Add Instance and you will be supplied with the command to run on your server. It will look like the following.

```bash
sudo cscli console enroll abcd1e2fg3456hi789jklmn0o
```

Run that command and approve it on the console.

## Bouncer Configuration

I had to manually add a bouncer for CrowdSec after installation to get a API key.

```bash
sudo cscli bouncers add haproxy
```

Your API key will be output as follows.

```bash
Api key for 'haproxy':

   091cc8a021dbda47884a8870b1b1dd8a

Please keep this key since you will not be able to retrieve it!
```

Copy the API key and paste it into ```/etc/crowdsec/bouncers/crowdsec-haproxy-bouncer.conf```.
You can also add the ban template path as below.

```bash
ENABLED=true
API_KEY=091cc8a021dbda47884a8870b1b1dd8a
# haproxy
# path to community_blocklist.map
MAP_PATH=/var/lib/crowdsec/lua/haproxy/community_blocklist.map
# bounce for all type of remediation that the bouncer can receive from the local API
BOUNCING_ON_TYPE=all
FALLBACK_REMEDIATION=ban
REQUEST_TIMEOUT=3000
UPDATE_FREQUENCY=10
# live or stream
MODE=stream
# exclude the bouncing on those location
EXCLUDE_LOCATION=
#those apply for "ban" action
# /!\ REDIRECT_LOCATION and RET_CODE can't be used together. REDIRECT_LOCATION take priority over RET_CODE
# path to ban template
BAN_TEMPLATE_PATH=/var/lib/crowdsec/lua/haproxy/templates/ban.html
REDIRECT_LOCATION=
RET_CODE=
#those apply for "captcha" action
# ReCaptcha Secret Key
SECRET_KEY=
# Recaptcha Site key
SITE_KEY=
# path to captcha template
CAPTCHA_TEMPLATE_PATH=/var/lib/crowdsec/lua/haproxy/templates/captcha.html
CAPTCHA_EXPIRATION=3600
```

## HAProxy Configuration

Under the global section you add the following

```bash
    # Crowdsec bouncer
    lua-prepend-path /usr/lib/crowdsec/lua/haproxy/?.lua
    lua-load /usr/lib/crowdsec/lua/haproxy/crowdsec.lua
    setenv CROWDSEC_CONFIG /etc/crowdsec/bouncers/crowdsec-haproxy-bouncer.conf
```

This is where my configuration is a bit fun. I have a loop in my HAProxy to allow for internal access to go directly to the servers and not through Cloudflare. See [this](https://ciphermenial.github.io/posts/my-haproxy-config/) post for an explanation of my HAProxy set up.

On my frontend that makes the decision on what frontend to send the connection to; I make sure only Cloudflare IP addresses are allowed to connect. This means no external connection can send a request from another IP and spoof the [CF-Connecting-IP](https://developers.cloudflare.com/fundamentals/get-started/reference/http-request-headers/#cf-connecting-ip) header.
You will need to create a file with the Cloudflare IPs. This is simple by doing the following.

```bash
sudo wget -O /etc/haproxy/CF_ips.lst https://www.cloudflare.com/ips-v4
```

The frontend used for redirecting the connection is as follows.

```bash
frontend https-redirect
    bind *:443
    mode tcp
    option tcplog
    tcp-request inspect-delay 5s
    tcp-request content accept if { req_ssl_hello_type 1 }
    acl internal src 192.168.88.1/24
    acl cloudflare src -f /etc/haproxy/CF_ips.lst
    use_backend cloudflare if cloudflare
    use_backend internal if internal
```

Make sure to set your internal IP ranges at `acl internal src`. You can add multiple ranges here by separating them with a space like `192.168.88.1/24 192.168.89.1/24`. You could alternatively do the same as the Cloudflare acl and save a list of internal IP ranges in a file.

Since I only want CrowdSec working on my external (cloudflare) frontend, I only place the bouncer configuration there.

```bash
frontend cloudflare
    bind *:7000 accept-proxy ssl crt domain.com.pem

    # CloudFlare CF-Connecting-IP header to source IP for Crowdsec decisions
    http-request set-src req.hdr(CF-Connecting-IP)

    # Crowdsec bouncer
    stick-table type ip size 10k expire 30m # declare a stick table to cache captcha verifications
    http-request lua.crowdsec_allow # action to identify crowdsec remediation
    http-request track-sc0 src if { var(req.remediation) -m str "captcha-allow" } # cache captcha allow decision
    http-request redirect location %[var(req.redirect_uri)] if { var(req.remediation) -m str "captcha-allow" } # redirect to initial url
    http-request use-service lua.reply_captcha if { var(req.remediation) -m str "captcha" } # serve captcha template if remediation is captcha
    http-request use-service lua.reply_ban if { var(req.remediation) -m str "ban" } # serve ban template if remediation is ban
```

The important part that differentiates from the [CrowdSec documentation](https://doc.crowdsec.net/docs/next/bouncers/haproxy) is setting the CF-Connecting-IP as the source IP.

The final part is adding the 2 required backends. The names of the backends must be `captcha_verifier` and `crowdsec`

```bash
# Backend for google to allow DNS resolution if using reCAPTCHA
backend captcha_verifier
    server captcha_verifier www.google.com:443 check

# Backend for crowdsec to allow DNS resolution
backend crowdsec
    server crowdsec localhost:8080 check
```

## Captcha Configuration

There is nothing special to do here and you can follow the [official documentation](https://doc.crowdsec.net/docs/next/bouncers/haproxy#setup-captcha) for this.

You should now have a working bouncer for HAProxy.
