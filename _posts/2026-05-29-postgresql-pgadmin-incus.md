---
title: Installing PostgreSQL and PGAdmin with Incus
categories: [Guides,PKI]
tags: [guides,incus,linux,debian,postgresql,lxc,postgresql,pgadmin4]
image:
  path: assets/img/title/postgresql-pgadmin-incus.svg
---
With my PostgreSQL setup, I have 2 LXC instances. One is PostgreSQL, and the other is for PGAdmin4 Web. I do this because I should learn the basics of SQL commands but I haven't.

## Launch Instances

> These days I only use [Debian](https://www.debian.org/) for my instances.
{: .prompt-info }

```bash
incus launch images:debian/trixie postgresql
incus launch images:debian/trixie pgadmin
```

## PostgreSQL Instance

### Install PostgreSQL

Go into the shell of the instance named **postgresql**.

```bash
incus exec postgresql bash
```

Install the base common package and run the script to add the PostgreSQL repositories.

```bash
apt install postgresql-common
/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

This will output the following.

```
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be trixie-pgdg.

Press Enter to continue, or Ctrl-C to abort.

Using keyring /usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg
Writing /etc/apt/sources.list.d/pgdg.sources ...

Running apt-get update ...
Hit:1 http://deb.debian.org/debian trixie InRelease
Hit:2 http://deb.debian.org/debian trixie-updates InRelease
Hit:3 http://deb.debian.org/debian-security trixie-security InRelease
Hit:4 https://apt.postgresql.org/pub/repos/apt trixie-pgdg InRelease
Reading package lists... Done                           

You can now start installing packages from apt.postgresql.org.

Have a look at https://wiki.postgresql.org/wiki/Apt for more information;
most notably the FAQ at https://wiki.postgresql.org/wiki/Apt/FAQ
```

You can now install PostgreSQL latest version. As of this post it is version 18.

```bash
apt install postgresql-18
```

The reason I add the repositories is for extra extensions I need for other services like Immich. The extension you need to install for that is `postgresql-18-pgvector`.

### Configure PostgreSQL

You will need to modify `/etc/postgresql/18/main/pg_hba.conf`{: .filepath} to allow access remotely to PostgreSQL. [Here](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html) is more information about this configuration file.

I add a section for IPv6 remote connections. The address I add is the IPv6 /64 assigned to my Incus network bridge. This example is using the [IPv6 address prefix reserved for documentation](https://datatracker.ietf.org/doc/html/rfc3849).

```
# Database administrative login by Unix domain socket
local   all             postgres                              peer

# TYPE  DATABASE        USER            ADDRESS               METHOD

# "local" is for Unix domain socket connections only
local   all             all                                   peer
# IPv4 local connections:
host    all             all             127.0.0.1/32          md5
# IPv6 local connections:
host    all             all             ::1/128               md5
# IPv6 remote connections:
host    all             all             2001:db8:1:b33f::/64  scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                   peer
host    replication     all             127.0.0.1/32          md5
host    replication     all             ::1/128               md5
```
{: file="/etc/postgresql/18/main/pg_hba.conf" }

Restart PostgreSQL to apply these changes.

## PGAdmin Instance

> This is following the directions from pgadmin for [installing using APT](https://www.pgadmin.org/download/pgadmin-4-apt/)
{: .prompt-info }

### Add PGAdmin Repository

First install the public key and add the repository configuration file. For this you will need to have curl and gpg installed. The latest debian images for Incus have curl installed by default.

```bash
apt update
apt install gpg
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list && apt update'
```

### Install PGAdmin4 Web

```bash
apt install pgadmin4-web
/usr/pgadmin4/bin/setup-web.sh
```

The `/usr/pgadmin4/bin/setup-web.sh`{: .filepath} script is straightforward. It asks you to enter an email address and password for login. Once completed, the web server is started.

### Configure PGAdmin4 Web Authentication

I use Keycloak as the only way to authenticate with PGAdmin4. I add this autentication source to `/usr/pgadmin4/web/config_local.py`

```python
##########################################################################
# External Authentication Sources
##########################################################################

# Default setting is internal
# External Supported Sources: ldap, kerberos, oauth2
# Multiple authentication can be achieved by setting this parameter to
# ['ldap', 'internal'] or ['oauth2', 'internal'] or
# ['webserver', 'internal'] etc.
# pgAdmin will authenticate the user with ldap/oauth2 whatever first in the
# list, in case of failure the second authentication option will be considered.

AUTHENTICATION_SOURCES = ['oauth2']

##########################################################################

##########################################################################
# OAuth2 Configuration
##########################################################################

# Multiple OAUTH2 providers can be added in the list like [{...},{...}]
# All parameters are required

OAUTH2_CONFIG = [
    {
        'OAUTH2_NAME': 'keycloak',
        'OAUTH2_DISPLAY_NAME': 'Keycloak',
        'OAUTH2_CLIENT_ID': 'pgadmin',
        'OAUTH2_CLIENT_SECRET': 'RandomSecret',
        'OAUTH2_SERVER_METADATA_URL': 'https://accounts.example.com/realms/master/.well-known/openid-configuration',
        'OAUTH2_TOKEN_URL': 'https://accounts.example.com/realms/master/protocol/openid-connect/token',
        'OAUTH2_AUTHORIZATION_URL': 'https://accounts.example.com/realms/master/protocol/openid-connect/auth',
        'OAUTH2_API_BASE_URL': 'https://accounts.example.com/realms/master',
        'OAUTH2_USERINFO_ENDPOINT': 'https://accounts.example.com/realms/master/protocol/openid-connect/userinfo',
        'OAUTH2_SCOPE': 'openid email profile',
        'OAUTH2_ICON': 'fa-brands fa-openid',
        'OAUTH2_BUTTON_COLOR': None,
        }
]

# After Oauth authentication, user will be added into the SQLite database
# automatically, if set to True.
# Set it to False, if user should not be added automatically,
# in this case Admin has to add the user manually in the SQLite database.

OAUTH2_AUTO_CREATE_USER = False

##########################################################################
```
{: file="/usr/pgadmin4/web/config_local.py" }

I manually add my account as an admin. You could manage in Keycloak if you had many people that needed to use this PGAdmin4 instance and change `OAUTH2_AUTO_CREATE_USER` to True.

You do this with the `/usr/pgadmin4/web/setup.py`{: .filepath}
You need to activate the venv for pgadmin first and then you can run the command to add an external user.

```bash
source /usr/pgadmin4/venv/bin/activate
python3 /usr/pgadmin4/web/setup.py add-external-user user@example.com --auth-source oauth2 --role Administrator --email user@example.com
```

You will now be able to sign in with your user from Keycloak.
