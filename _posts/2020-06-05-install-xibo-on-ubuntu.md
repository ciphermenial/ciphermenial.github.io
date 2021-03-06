---
layout: post
title:  Install Xibo on Ubuntu 20.04
categories: [Server,Linux,Install]
excerpt: This guide shows how to install Xibo CMS 2.3.3 onto Ubuntu 20.04 LTS
---

# Introduction
I was recently testing out Xibo for use within a school. I origianlly used the Docker install but ended up feeling more comfortable with a manual installation. I have installed it on Ubuntu Server 20.04 using Apache2 as the web server and enforcing TLS. I have placed the necessary certificate and key file under /etc/ssl/private. You will need to follow another guide for creating the required certificates. I recommend using Let's Encrypt with their certbot.

## Notes
* The MariaDB and PHP versions installed here are not currently supported by Xibo.
* The library location is /var/www/Library due to the installer writing the install log file to /var/www/library. The installer requires that the library directory be empty.
* A local install of QuickChart is not covered in this guide but recommended.

# Install Requirements
All of the requirements can be installed using apt.

```bash
sudo apt install mariadb-server mariadb-client apache2 php php-cli php-gd php-json php-dom php-mysql php-zip php-soap php-curl php-xml php-mbstring php-zmq libapache2-mod-xsendfile
```

# Install Xibo CMS
When I am configuring server software outside of a package manager I always place it under the /srv folder. I will be installing Xibo under /srv/xibo-cms.

```bash
sudo mkdir /srv/xibo-cms
cd /srv/xibo-cms
sudo wget https://github.com/xibosignage/xibo-cms/releases/download/2.3.3/xibo-cms-2.3.3.tar.gz
```

This extracts the contents of the archive without placing it into a folder

```bash
sudo tar -xvzf xibo-cms-2.3.3.tar.gz --strip-components=1
sudo rm xibo-cms-2.3.3.tar.gz
```

The apache2 user 'www-data' needs to be set as owner of all the extracted items.

```bash
sudo chown -R www-data:www-data /srv/xibo-cms
```

This deletes the existing /var/www directory and creates a symlink to /srv/xibo-cms.

```bash
sudo rm -r /var/www
sudo ln -s /srv/xibo-cms /var/www
```

## Configure Apache2
This enables the necessary apache2 modules and creates a site configuration using vim.

```bash
sudo a2enmod rewrite
sudo a2enmod ssl
sudo a2enmod session
sudo vim /etc/apache2/sites-available/xibo-cms.conf
```

### xibo-cms.conf

```apache
<VirtualHost *:80>
    DocumentRoot "/var/www/web"
    ServerName xibo.domain.com
    XSendFile on
    XSendFilePath /var/www/Library
    <Directory "/var/www/web">
        AllowOverride All
        Options Indexes FollowSymLinks MultiViews
        Order allow,deny
        Allow from all
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:443>
    DocumentRoot "/var/www/web"
    ServerName xibo.domain.com
    XSendFile on
    XSendFilePath /var/www/Library
    SSLEngine on
    SSLCertificateFile "/etc/ssl/private/ssl-cert.pem"
    SSLCertificateKeyFile "/etc/ssl/private/ssl-private-key.pem"
    <Directory "/var/www/web">
        AllowOverride All
        Options Indexes FollowSymLinks MultiViews
        Order allow,deny
        Allow from all
        Require all granted
    </Directory>
</VirtualHost>
```

This disables the default site and enables the newly created xibo-cms site configuration.

```bash
sudo a2dissite 000-default.conf
sudo a2ensite xibo-cms.conf
```

## Configure MariaDB
This configures a root password for MariaDB. Make sure to change MY_NEW_PASSWORD to your password of choice.

```bash
sudo mysql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MY_NEW_PASSWORD';
FLUSH PRIVILEGES;
quit
```

## Configure PHP
PHP configuration needs to be modified to allow upload of larger files.

```bash
sudo vim /etc/php/7.4/apache2/php.ini
```

Modify the following lines as shown

```
max_execution_time = 300
memory_limit = 256M
post_max_size = 2G
upload_max_filesize = 2G
session.cookie_secure = Off
session.cookie_httponly = On
session.cookie_samesite = Lax
```

## Configure XMR
### Create XMR Configuration File

```bash
sudo vim /srv/xibo-cms/vendor/xibosignage/xibo-xmr/bin/config.json
```

Enter the following information and change the pubOn IP address to the public IP of the server.

```json
{
    "listenOn": "tcp://127.0.0.1:50001",
    "pubOn": ["tcp://192.168.1.1:9505"],
    "debug": false
}
```

Set www-data as the owner of the file.

```bash
sudo chown www-data:www-data /srv/xibo-cms/vendor/xibosignage/xibo-xmr/bin/config.json
```

### Create XMR service

```bash
sudo vim /etc/systemd/system/xibo-xmr.service
```

Enter the following.

```
[Unit]
Description=Xibo XMR
After=network.target

[Service]
User=www-data
Group=www-data
ExecStart=/usr/bin/php /srv/xibo-cms/vendor/bin/xmr.phar
Restart=always
KillMode=process
RestartSec=1

[Install]
WantedBy=multi-user.target
```

Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable xibo-xmr.service
sudo systemctl start xibo-xmr.service
```

Check the status of the xibo-xmr service to confirm it is working.

```bash
sudo systemctl status xibo-xmr.service
```

## Configure XTR

```bash
sudo crontab -u www-data -e
```

Select the editor you prefer and then enter the following line.

```
* * * * * /usr/bin/php /var/www/bin/xtr.php
```

# Configure Firewall

```bash
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw allow 9505/tcp
sudo ufw enable
```

# Complete Installation

```bash
sudo systemctl restart apache2
```

You can now browse to the server at the location you configured in the xibo-cms.conf

# Upgrading

## Backup

The simplest thing to do is to stop apache2 and xibo-xmr service, moving the /srv/xibo-cms directory, and backing up the database.

```bash
sudo systemctl stop apache2 xibo-xmr
sudo mv /srv/xibo-cms /srv/xibo-cms.backup
sudo mysqldump xibo-cms > xibo-cms.sql
```

Create xibo-cms directory and change to it. Download the new version, in this example 2.3.4 extract it and copy back the necessary files. You will also need to delete the install/index.php file to stop a warning from appearing.

## Installation

```bash
sudo mkdir /srv/xibo-cms
cd /srv/xibo-cms
sudo wget https://github.com/xibosignage/xibo-cms/releases/download/2.3.4/xibo-cms-2.3.4.tar.gz
sudo tar -xvzf xibo-cms-2.3.4.tar.gz --strip-components=1
sudo cp /srv/xibo-cms.backup/web/settings.php web/
sudo cp -r /srv/xibo-cms.backup/Library .
sudo cp /srv/xibo-cms.backup/vendor/xibosignage/xibo-xmr/bin/config.json vendor/xibosignage/xibo-xmr/bin/
sudo chown -R www-data:www-data /srv/xibo-cms
sudo rm web/install/index.php
```

Start up the services again.

```bash
sudo systemctl start apache2 xibo-xmr
```

After you have logged back in you may see no Layouts. Press Shift+F5 to clear cache on your browser and reload the page.
