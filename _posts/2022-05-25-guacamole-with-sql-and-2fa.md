---
title:  Install Guacamole on Ubuntu
categories: [Guides,Guacamole]
tags: [guides,ubuntu,linux,guacamole,apache,tomcat,totp,mariadb]
image:
  path: /assets/img/title/guacamole-with-sql-and-2fa.svg
---

This was done to provide remote access to a server and make it as secure as possible.

## Notes
- This install was done in an LXD container behind a HAProxy reverse proxy.
- This is only a secure install if running behind a reverse proxy and is only accessible from the reverse proxy server/s.
- In this example I have only installed the requirements for RDP and SSH connections.

# Install Requirements
All of the requirements can be installed using apt.

```bash
apt install gcc libcairo2-dev default-jdk make libjpeg-dev libtool-bin \
libossp-uuid-dev freerdp2-dev libssh2-1-dev libpango1.0-dev mariadb-server
```

# Install Tomcat 9.0.72

> [Guacamole is not compatible with Tomcat 10](https://issues.apache.org/jira/browse/GUACAMOLE-1325)
{: .prompt-danger }

When I am configuring server software outside of a package manager I always place it under the /srv folder. I will be installing Tomcat and Guacamole under /srv/tomcat.

```bash
cd /srv
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.72/bin/apache-tomcat-9.0.72.tar.gz
```

This extracts the contents of the archive and places it into a directory named tomcat.

```bash
mkdir /srv/tomcat
tar -xzf apache-tomcat-9.0.72.tar.gz --strip-components=1 -C /srv/tomcat
rm apache-tomcat-9.0.72.tar.gz
```

Now you will need to create a systemd service for autostarting Tomcat.

```bash
vim /etc/systemd/system/tomcat.service
```

## tomcat.service file contents

```
[Unit]
Description=Tomcat 9 servlet container
After=network.target

[Service]
Type=forking

Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"
Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom -Djava.awt.headless=true"

Environment="CATALINA_BASE=/srv/tomcat"
Environment="CATALINA_HOME=/srv/tomcat"
Environment="CATALINA_PID=/srv/tomcat/temp/tomcat.pid"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"

ExecStart=/srv/tomcat/bin/startup.sh
ExecStop=/srv/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```

Now you will need to reload systemd manager configuration and enable the Tomcat service.

```bash
systemctl daemon-reload
systemctl enable --now tomcat
```

# Install Guacamole Server 1.5.0
I use git to clone the latest version. You can find out exactly what all of these commands are doing by reading the [official Guacamole documentation](https://guacamole.apache.org/doc/gug/installing-guacamole.html).

```bash
cd
git clone git://github.com/apache/guacamole-server.git
cd guacamole-server
autoreconf -fi
./configure --with-init-dir=/etc/init.d
make
make install
ldconfig
```

You will need to reload the systemd manager configuration again and start guacd.

```bash
systemctl daemon-reload
sudo systemctl start guacd
sudo systemctl enable guacd
```

# Install Guacamole Client 1.5.0

```bash
mkdir /srv/guacamole
cd /srv/guacamole
wget https://downloads.apache.org/guacamole/1.5.0/binary/guacamole-1.5.0.war -O guacamole.war
ln -s /srv/guacamole/guacamole.war /srv/tomcat/webapps
ln -s /srv/guacamole /etc/guacamole
vim guacamole.properties
```

## guacamole.properties file contents
The mysql and totp entries are for once it is completely configured. You can set the mysql user and password to the same later when creating the database. I like to put the totp-issuer name in so it shows something more understandable for your install than the default. I had to set guacd-hostname to 127.0.0.1 because otherwise it would listen on IPv6 only and failed to work. The alternative is settings the guacd proxy parameters for every connections to point to ::1 as the address.

```bash
guacd-hostname: 127.0.0.1
guacd-port:    4822

auth-provider: net.sourceforge.guacamole.net.auth.mysql.MySQLAuthenticationProvider

mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole
mysql-username: guacamole
mysql-password: PASSWORD
mysql-driver: mariadb

totp-issuer: Guacamole
```

# Install Extensions and Libraries

## Install MariaDB Extension

```bash
cd
wget https://dlcdn.apache.org/guacamole/1.5.0/binary/guacamole-auth-jdbc-1.5.0.tar.gz
mkdir /srv/guacamole/extensions
tar -xzf guacamole-auth-jdbc-1.5.0.tar.gz
rm guacamole-auth-jdbc-1.5.0.tar.gz
cp guacamole-auth-jdbc-1.5.0/mysql/guacamole-auth-jdbc-mysql-1.4.0.jar /srv/guacamole/extensions
```

## Install TOTP Extension for 2FA

```bash
wget https://dlcdn.apache.org/guacamole/1.5.0/binary/guacamole-auth-totp-1.5.0.tar.gz
tar -xzf guacamole-auth-totp-1.5.0.tar.gz
rm guacamole-auth-totp-1.5.0.tar.gz
cp guacamole-auth-totp-1.5.0/guacamole-auth-totp-1.5.0.jar /srv/guacamole/extensions
```

## Install MariaDB Library

> Guacamole does not work with version 3.0.5 of the connector
{: .prompt-danger }

```bash
mkdir /srv/guacamole/lib
cd /srv/guacamole/lib
wget https://downloads.mariadb.com/Connectors/java/connector-java-2.7.8/mariadb-java-client-2.7.8.jar
```

# Create Guacamole Database

Enter the mariadb console by running `mariadb`

```sql
CREATE DATABASE guacamole;
CREATE USER 'guacamole' IDENTIFIED by 'PASSWORD';
GRANT SELECT,INSERT,UPDATE,DELETE on guacamole.* to 'guacamole';
FLUSH PRIVILEGES;
QUIT;
```

## Populate The Database

```bash
cd
cat guacamole-auth-jdbc-1.5.0/mysql/schema/*.sql | mariadb guacamole
```

# Finishing Up
I wanted Tomcat to automatically redirect to /guacamole rather than loading the default Tomcat landing page. To do this you need to replace the index.jsp file located under /srv/tomcat/webapps/ROOT/ with the following.

```bash
cd /srv/tomcat/webapps/ROOT
mv index.jsp index.jsp.old
vim index.jsp
```

and add the line

```jsp
<% response.sendRedirect("/guacamole"); %>
```

I am also running this behind a HAProxy and wanted to configure X-Forwarded-For to present the correct client IP in the logs on Guacamole. To do this you need to add some lines to the Tomcat server.xml file.

```bash
cd /srv/tomcat/conf
vim server.xml
```

I added the following above the other Valve section in server.xml

```xml
        <!-- RemoteIp valve, process X-Forwarded-For headers
             Documentation at: /docs/config/valve.html -->
        <Valve className="org.apache.catalina.valves.RemoteIpValve"
               remoteIpHeader="x-forwarded-for"
               remoteIpProxiesHeader="x-forwarded-by"
               protocolHeader="x-fowarded-proto"
               internalProxies="127.0.0.1" />
```

Finally restart the services

```bash
systemctl restart tomcat guacd
```

You should now be able to connect to your Guacamole install by going to ServerIP:8080

> The default username and password are guacadmin/guacadmin
{: .prompt-info}
