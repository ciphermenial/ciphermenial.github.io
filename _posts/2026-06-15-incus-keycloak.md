---
title: Installing & Configuring Keycloak
categories: [Guides,Incus]
tags: [guides,incus,linux,debian,postgresql,lxc,keycloak,saml,ldap,oidc,sso]
image:
  path: assets/img/title/incus-keycloak.svg
---

apt update
apt install openjdk-25-jdk-headless unzip
curl -OJL https://github.com/keycloak/keycloak/releases/download/26.6.3/keycloak-26.6.3.zip
unzip keycloak-26.6.3.zip && mv keycloak-26.6.3 /srv/keycloak
vim /etc/profile.d/keycloak.sh

```bash
export JAVA_OPTS="-XX:MetaspaceSize=96M -XX:MaxMetaspaceSize=256m -Dfile.encoding=UTF-8 -Dsun.stdout.encoding=UTF-8 -Dsun.err.encoding=UTF-8 -Dstdout.encod
ing=UTF-8 -Dstderr.encoding=UTF-8 -XX:+ExitOnOutOfMemoryError -Djava.security.egd=file:/dev/urandom -XX:+UseG1GC -XX:FlightRecorderOptions=stackdepth=512 -
Djava.net.preferIPv6Addresses=true -Djava.net.preferIPv4Stack=false"
```
{: file="/etc/profile.d/keycloak.sh" }

echo $JAVA_OPTS
vim /srv/keycloak/conf/keycloak.conf

```ini
# Basic settings for running in production. Change accordingly before deploying the server.

# Database

# The database vendor.
db=postgres

# The username of the database user.
db-username=keycloak

# The password of the database user.
db-password=RandomPassword

# The full database JDBC URL. If not provided, a default URL is set based on the selected database vendor.
db-url=jdbc:postgresql://postgresql.pri.incus/keycloak

# Observability

# If the server should expose healthcheck endpoints.
#health-enabled=true

# If the server should expose metrics endpoints.
#metrics-enabled=true

# Hostname for the Keycloak server.
hostname=https://accounts.example.com
hostname-backchannel-dynamic=false

https-port=443
https-certificate-file=/etc/ssl/keycloak/keycloak-server.crt.pem
https-certificate-key-file=/etc/ssl/keycloak/keycloak-server.key.pem

log=file
log-file=/var/log/keycloak/keycloak.log
log-level=INFO


proxy-headers=xforwarded
proxy-trusted-addresses=10.10.10.254,[2001:0db8:85a3:0000:0000:8a2e:0370:7334]
```
{: file="/srv/keycloak/conf/keycloak.conf }

bin/kc.sh build
bin/kc.sh start --optimized
vim /etc/systemd/system/keycloak.service

```ini
[Unit]
Description=Keycloak Server
After=syslog.target network.target
Before=httpd.service

[Service]
ExecStart=/srv/keycloak/bin/kc.sh start
WorkingDirectory=/srv/keycloak
SuccessExitStatus=0 143

[Install]
WantedBy=multi-user.target
```
{: file="/etc/systemd/system/keycloak.service" }

systemctl daemon-reload
systemctl enable keycloak
systemctl start keycloak
