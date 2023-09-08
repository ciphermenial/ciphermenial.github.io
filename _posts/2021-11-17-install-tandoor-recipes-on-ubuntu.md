---
title:  Install Tandoor Recipes in Ubuntu
categories: [Guides,Tandoor Recipes]
tags: [guides,tandoor recipes,linux,ubuntu,lxd,postgresql]
image: 
  path: /assets/img/title/install-tandoor-recipes-on-ubuntu.svg
---

In this guide we will look at manually installing [Tandoor Recipes](https://docs.tandoor.dev/). I have done this in a LXD container but you should be able to follow it using sudo commands for a normal install.

### Notes
- This install was done in a LXD container.
- I have a haproxy running in front of this.
- I already have a postgresql server in place and built a database named *recipes* with a user with the same name.
- I installed Node.js from [NodeSource](https://github.com/nodesource/distributions/blob/master/README.md)

## Installation
The requirements can be installed using apt and npm. I also am creating a user for this app called recipes.
First I install Node.js.

```bash
apt update
apt install -y ca-certificates curl gnupg
mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | tee /etc/apt/sources.list.d/nodesource.list
apt update
apt install nodejs -y
```

Now you can install the other requirements.

```bash
apt install gettext libjpeg9 libwebp7 build-essential musl-dev cargo python3-venv python3-dev libldap2-dev libssl-dev libsasl2-dev nginx python-is-python3
npm install yarn -g
useradd recipes
```

I didn't want to have all the files from the github repository in my web servers directory so I originally pulled it down to the home direcotry.

```bash
cd ~
git clone https://github.com/TandoorRecipes/recipes.git
```

I created a new directory for the service. I always do this under the /srv directory. I copied across the parts I believe are necessary.

```bash
mkdir /srv/recipes
cd /srv/recipes
mkdir mediafiles
cp -r ~/recipes/requirements.txt ~/recipes/cookbook/ ~/recipes/recipes/ ~/recipes/vue ~/recipes/makemessages.cmd ~/recipes/manage.py ~/recipes/yarn.lock .
cp ~/recipes/.env.template .env
```

You will need to now edit the .env file you copied over and change the following sections. Parts in uppercase need to be changed.

### .env file changes
```bash
SECRET_KEY=SOMETHING_RANDOM

# your default timezone See https://timezonedb.com/time-zones for a list of timezones
TIMEZONE=COUNTRY/AREA

DB_ENGINE=django.db.backends.postgresql
POSTGRES_HOST=YOUR_POSTGRESQL_SERVER_HOSTNAME_OR_IP
POSTGRES_PORT=5432
POSTGRES_USER=recipes
POSTGRES_PASSWORD=YOUR_PASSWORD
POSTGRES_DB=recipes
```

Now create a Python virtual env. and install the requirements.

```bash
python -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt 
```

Now install the frontend requirements and build them.

```bash
cd vue
yarn install
yarn build
```

Fill the database and collect the static files.

```bash
cd ..
python manage.py migrate
python manage.py collectstatic_js_reverse
python manage.py collectstatic
```

Change ownership of the recipes directory and give appropriate access to the mediafiles directory.

```bash
chown -R recipes:www-data /srv/recipes
chmod -R 755 mediafiles
```

Now you need to create a systemd service for gunicorn. Create and modify a new service file /etc/systemd/system/recipes.service

### recipes.server file contents

```bash
[Unit]
Description=gunicorn daemon for recipes
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=3
User=recipes
Group=www-data
WorkingDirectory=/srv/recipes
EnvironmentFile=/srv/recipes/.env
ExecStart=/srv/recipes/venv/bin/gunicorn --error-logfile /tmp/gunicorn_err.log --log-level debug --capture-output --bind unix:/srv/recipes/recipes.sock recipes.wsgi:application

[Install]
WantedBy=multi-user.target
```

You will also need to place the following into the /etc/nginx/sites-available/default file. I clear it out and replace it completely. If you don't have a reverse proxy in front of this server you need to add *proxy_set_header X-Forwarded-Proto $scheme;* under the location section.

```nginx
server {
    listen 8002;
    listen [::]:8002;
	
    root /srv/recipes/;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Serve media files
    location /media/ {
        alias /srv/recipes/mediafiles/;
    }

    location / {
        proxy_set_header Host $http_host;
        proxy_pass http://unix:/srv/recipes/recipes.sock;
    }
}
```

Now enable and restart the necessary services and you should be good to go.

```bash
systemctl daemon-reload
systemctl enable recipes
systemctl start recipes
systemctl restart nginx
```

## Upgrade
Pull the latest changes for github and copy the necessary files to the install.

```bash
cd ~/recipes
git pull
cd /srv/recipes
cp -r ~/recipes/requirements.txt ~/recipes/cookbook/ ~/recipes/recipes/ ~/recipes/vue ~/recipes/makemessages.cmd ~/recipes/manage.py ~/recipes/yarn.lock .
```

Activate the python venv and run update commands.

```bash
source venv/bin/activate
pip install -r requirements.txt
cd vue
yarn install
yarn build
cd ..
python manage.py migrate
python manage.py collectstatic_js_reverse
python manage.py collectstatic
```

Restart the recipes service and you're done.

```bash
systemctl restart recipes
```

## Upgrade to Ubuntu 22.04 LTS
I had an issue after upgrading to 22.04 LTS because it went from Python 3.9 to Python 3.10.
After the upgrade I had to delete and rebuild the Python venv. For some reason I also had to install venv again.

```bash
apt install python3-venv
cd /srv/recipes
rm -r venv/
rm staticfiles/staticfiles.json
python -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cd vue
yarn install
yarn build
cd ..
python manage.py collectstatic_js_reverse
python manage.py collectstatic
systemctl start recipes
```
