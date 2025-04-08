---
title: Install MariaDB on Debian Bookworm
categories: [Guides,Incus]
tags: [guides,incus,linux,lxc,debian,mariadb]
image:
  path: assets/img/title/install-mariadb-debian.svg
---
This is part of my [Home Lab](/homelab/) guides. I am installing [MariaDB](https://mariadb.com/) on [Debian Bookworm](https://www.debian.org/releases/bookworm/) running inside a [LXC](https://linuxcontainers.org/lxc/introduction/) container managed by [Incus](https://linuxcontainers.org/incus/introduction/).

## Install MariaDB
First we will launch a Linux Container running the default Debian Bookworm image named **mariadb**. Then we will switch to a shell in that container.

```bash
incus launch images:debian/12 mariadb
incus exec mariadb bash
```

Now we will install MariaDB Server in the container.
```bash
apt update
apt install mariadb-server
```

## Secure MariaDB
At this point you want to secure MariaDB.

```bash
mariadb-secure-installation
```

This outputs the following:
```
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n]
Enabled successfully!
Reloading privilege tables..
 ... Success!
```

Here is an explanation of [unix_socket authentication](https://mariadb.com/kb/en/authentication-plugin-unix-socket/). It doesn't matter what you do here as unix_socket authentication is enabled by default.

```
You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.
```

You don't need to change the root password as you are using unix_socket authentication.

```
By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] 
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] 
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] 
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] 
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

With the rest of the wizard you will accept the defaults and restart the mariadb service `systemctl restart mariadb.server`.

You have now configured the MariaDB container for use with your other containers.

## Useful Commands
### Access Console
To login to the MariaDB console with the root users you simply type:

```bash
mariadb
```

You will be presented with the following where you can enter SQL statements.

```
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 31
Server version: 10.11.11-MariaDB-0+deb12u1 Debian 12

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

## Useful Commands

`CREATE DATABASE example;` will create a database named **example**.

This will create a user with the same name as the database and password **kd437ePJ!m6HaD3MCB$hNnW&**. The % means the user can sign in from any other system (use localhost if you only need localhost access).

`CREATE USER example@'%' IDENTIFIED BY 'kd437ePJ!m6HaD3MCB$hNnW&';`

We will then grant full access to the **example** database.

`GRANT ALL PRIVILEGES ON example.* TO example@'%';`

Reload all privileges from the privilege tables in the **mysql** database to apply these changes.

`FLUSH PRIVILEGES;`
