---
title: Using Ansible with Incus Containers
categories: [Guides,Incus]
tags: [guides,incus,linux,debian,ansible,lxc,keycloak]
image:
  path: assets/img/title/incus-ansible.svg
---
I recently spent far too much time learning the basics of [Ansible](https://docs.ansible.com/) to use apt to update all my LXC instances on my [Incus](https://linuxcontainers.org/incus/) servers. I am still learning a lot of the basics of Ansible, so if you see anything silly let me know!

## Launch Ansible Container
First I created a container to be the Ansible control node. Then entered the shell of that container.

```bash
incus launch images:debian/trixie ansible
incus shell ansible
```

## Prerequisites

### Install Incus Client

For managing Incus instances I needed the incus client installed. I am installing it using the [Zabbly](https://github.com/zabbly/incus) stable repository.

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

Next I added the Incus remote connections to my 2 Incus servers. I use Keycloak for auth using OIDC. I set the Incus configuration on my hosts to it to IPv6 on port 8443. I also added the oidc configuration for authentication.

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

After running each of these commands I am presented with a link to sign into my Keycloak server. After I signed in with that link, I returned to the CLI and after a moment it completed.

I checked the remotes were added with `incus remote list` which output something like the following.

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

[Here are the instructions](https://docs.ansible.com/projects/ansible/latest/installation_guide/intro_installation.html#pipx-install) for installing Ansible with [pipx](https://pipx.pypa.io/stable/) from the doco.

```bash
apt install pipx
pipx ensurepath
pipx install --include-deps ansible
```

## Configuring Ansible

I created a folder named ansible in the root directory and change to it. Then I created and edited `ansible.cfg`{: .filepath} with vim.

```bash
mkdir ansible
cd ansible
vim ansible.cfg
```

I originally copied a generic configuration from somewhere and ended up needing to set it as follows.

```ini
[defaults]
interpreter_python = /usr/bin/python3
timeout = 30
forks = 10

[inventory]
enable_plugins = community.general.incus, yaml
```
{: file="ansible/ansible.cfg" }

I add `interpreter_python = /usr/bin/python3` to stop a warning about the python version discovered because it was python3.13 and that could change.
`enable_plugins = community.general.incus, yaml` is added for obvious reasons.

### Creating the Inventory

Inventory is all the hosts you want to run playbooks against. I created an inventory directory and added 3 inventory yaml files.

```bash
mkdir inventory
cd inventory
touch hosts.yaml incus1.incus.yaml incus2.incus.yaml
```

> The Incus inventory plugin requires the inventory file to include incus.yaml or incus.yml in the name, otherwise it ignores it with the message:
> `Skipping due to inventory source not ending in "incus.yaml" nor "incus.yml"`
{: .prompt-warn }

The contents of the incus.yaml files are as follows.

```yaml
plugin: community.general.incus
remotes:
  - incus1
host_domain: pri.incus
host_fqdn: false
filters:
  - status=running
groups:
  lxc: "'squashfs' in ansible_incus_config['image.type']"
```
{: file="ansible/inventory/incus1.incus.yaml" }

```yaml
plugin: community.general.incus
remotes:
  - incus2
host_domain: sec.incus
host_fqdn: false
filters:
  - status=running
groups:
  lxc: "'squashfs' in ansible_incus_config['image.type']"
```
{: file="ansible/inventory/incus2.incus.yaml" }

The lxc group is because I didn't want to run playbooks against any OCI instances I have. The inbuilt filter options for the Incus inventory plugin uses Incus's builtin filter which only identifies containers and virtual machines. It does not seperate OCI and LXC instances, they are both identified as containers. That meant I had to find another way to filter the containers based on other output. I ran `ansible-inventory -i inventory --host` against a host of each different Incus instances and found **image.type** to be the best output to select from.

These are the results of checking against each instance type.

- Virtual machines = disk-kvm.img
- LXC = squashfs
- OCI = oci

That way I can create a group for virtual machines by adding the line `vms: "'disk-kvm.img' in ansible_incus_config['image.type']"`

I checked that it was working with the command `ansible-inventory -i inventory --graph`

This output something like the following.

```bash
@all:
  |--@ungrouped:
  |  |--localhost
  |--@incus:
  |  |--@incus_incus1:
  |  |  |--@incus_incus1_default:
  |  |  |  |--ansible.pri.incus
  |  |  |  |--forgejo.pri.incus
  |  |  |  |--guacamole.pri.incus
  |  |  |  |--haproxy.pri.incus
  |  |  |  |--immich.pri.incus
  |  |  |  |--keycloak.pri.incus
  |  |  |  |--lldap.pri.incus
  |  |  |  |--mariadb.pri.incus
  |  |  |  |--pgadmin.pri.incus
  |  |  |  |--phpmyadmin.pri.incus
  |  |  |  |--postgresql.pri.incus
  |  |  |  |--smtp.pri.incus
  |  |--@incus_incus2:
  |  |  |--@incus_incus2_default:
  |  |  |  |--grafana.sec.incus
  |  |  |  |--immich-ml.sec.incus
  |  |  |  |--loki.sec.incus
  |  |  |  |--prometheus.sec.incus
  |  |  |  |--debian.sec.incus
  |--@lxc:
  |  |--ansible.pri.incus
  |  |--forgejo.pri.incus
  |  |--guacamole.pri.incus
  |  |--haproxy.pri.incus
  |  |--keycloak.pri.incus
  |  |--mariadb.pri.incus
  |  |--pgadmin.pri.incus
  |  |--phpmyadmin.pri.incus
  |  |--postgresql.pri.incus
  |  |--smtp.pri.incus
  |  |--prometheus.sec.incus
```
