---
title: Using Ansible with Incus Containers
categories: [Guides,Incus]
tags: [guides,incus,linux,debian,ansible,lxc,keycloak]
image:
  path: assets/img/title/incus-ansible.svg
---
I recently spent far too much time learning the basics of [Ansible](https://docs.ansible.com/) to use apt to update all my LXC instances on my [Incus](https://linuxcontainers.org/incus/) servers. I am still learning a lot of the basics of Ansible, so if you see anything silly let me know!

## Launch Ansible Container
First create a container to be the Ansible control node. Then enter the shell of that container.

```bash
incus launch images:debian/trixie ansible
incus shell ansible
```

## Prerequisites

### Install Incus Client

For managing Incus instances you will need the incus client installed. I am installing it using the [Zabbly](https://github.com/zabbly/incus) stable repository.

```bash
curl -fsSL https://pkgs.zabbly.com/key.asc -o /etc/apt/keyrings/zabbly.asc
sh -c 'cat <<EOF > /etc/apt/sources.list.d/zabbly-incus-stable.sources
Enabled: yes
Types: deb
URIs: https://pkgs.zabbly.com/incus/stable
Suites: $(. /etc/os-release && echo ${VERSION_CODENAME})
Components: main
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/zabbly.asc

EOF'
apt update
apt install incus-client
```

### Add Incus Remotes

Next I added the Incus remote connections to my 2 Incus servers. I use Keycloak for auth using OIDC. You will need to set the incus configuration on your hosts. I set it to all IPv6 on port 8443. I also add the oidc configuration.

```bash
incus config set core.https_address='[::]:8443'
incus config set oidc.client.id='incus'
incus config set oidc.issuer='https://accounts.example.com/realms/master'
```

> I will be adding a post about installing and configuring Keycloak sometime soon.
{: .prompt-info }

Note that in my example the 2 servers are registered in local DNS with an internal domain.
- incus1.internal.example.com
- incus2.internal.example.com

```bash
incus remote add incus1 incus1.internal.example.com --auth-type oidc
incus remote add incus2 incus2.internal.example.com --auth-type oidc
```

This gives you a link to go to and sign into your OIDC server. After you have signed in with that link, return to the CLI and after a moment it should complete.

To check you can view the remotes added with `incus remote list` which will output something like this.

```
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
|      NAME       |                      URL                 |   PROTOCOL    |  AUTH TYPE  | PUBLIC | STATIC | GLOBAL |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| images          | https://images.linuxcontainers.org       | simplestreams | none        | YES    | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| incus1          | https://incus1.internal.example.com:8443 | incus         | oidc        | NO     | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| incus2          | https://incus2.internal.example.com:8443 | incus         | oidc        | NO     | NO     | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
| local (current) | unix://                                  | incus         | file access | NO     | YES    | NO     |
+-----------------+------------------------------------------+---------------+-------------+--------+--------+--------+
```

### Install Ansible

[Here are the instructions](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html#pipx-install) for installing Ansible with from the doco.

```bash
apt install pipx
pipx ensurepath
pipx install --include-deps ansible
```

## Configuring Ansible

