---
title: Adding OAuth to Paperless-ngx
categories: [Article,Paperless]
tags: [ubuntu,linux,paperless-ngx,haproxy,django,oauth]
image:
  path: /assets/img/title/oauth-paperless-ngx.svg
---

> Paperless-ngx now has inbuilt support for this, so you can ignore this entirely.
{: .prompt-info }

When I am deciding on web services to use on my home lab, I lean towards ones that support [SSO](https://en.wikipedia.org/wiki/Single_sign-on). I do this because I have Keycloak configured and working with my Yubikeys for [MFA](https://en.wikipedia.org/wiki/Multi-factor_authentication).

The problem here is I recently decided to try [Paperless-ngx](https://paperless-ngx.readthedocs.io) since so many people use it in their home labs. It does not have SSO support. However it is built with [django](https://www.djangoproject.com). I have previously configured [Tandoor Recipes](https://docs.tandoor.dev) in my home lab and it also uses django and supports SSO. It supports [OAuth](https://oauth.net) by using the project [django-allauth](https://django-allauth.readthedocs.io).

I thought I would see how that all fit together and try to make django-allauth work with Paperless-ngx. I have managed to make it work (it's not pretty though) and this is what bits I changed.

## requirements.txt
The first thing you need to do is make sure django-allauth is installed so Paperless-ngx can access it. To do this I added it to the requirements.txt with the line ```django-allauth==0.54.0```
Then I switched to the python venv and reran ```pip install -r requirements.txt```. You can simply switch to the venv and install django-allauth instead.

## settings.py
Now I needed to make modifications to ```src/paperless/settings.py``` because currently Paperless-ngx has no idea about django-allauth.
The first part I added was right at the start. I added ```import ast``` because that is required for a part I used from Tandoor Recipes. I don't know if it is required because I have no idea what I am doing.

You also need to add allauth to the INSTALLED_APPS section as follows.

```diff
    "guardian",
+    "allauth",
+    "allauth.account",
+    "allauth.socialaccount",
    *env_apps,
```

One of the issues I had was that I have a reverse proxy doing TLS offloading. This caused a problem with django-allauth. I'm not exactly sure what the problem was but when it tried to auth to my Keycloak server it didn't even reach it and redirected back to the Paperless-ngx login. To resolve that I needed to set up settings to make django aware of the reverse proxy.

```python
# Reverse Proxy requirements
REVERSE_PROXY_AUTH = __get_boolean("PAPERLESS_REVERSE_PROXY_AUTH")

if REVERSE_PROXY_AUTH:
    USE_X_FORWARDED_HOST = True
    SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
 ```
This will configure these parameters if PAPERLESS_REVERSE_PROXY_AUTH is set to true in the paperless.conf.

```python
# Enable Django Allauth
ENABLE_ALLAUTH = __get_boolean("PAPERLESS_ENABLE_ALLAUTH")
```
This sets the variable ENABLE_ALLAUTH to True or False based on what PAPERLESS_ENABLE_ALLAUTH is set to in the paperless.conf

```python
# Set Defualt access for Django Allauth
ALLAUTH_DEFAULT_ACCESS = __get_boolean("PAPERLESS_ALLAUTH_DEFAULT_ACCESS")
ALLAUTH_DEFAULT_GROUP = os.getenv('PAPERLESS_ALLAUTH_DEFAULT_GROUP', 'guest')
```
This was another part taken from Tandoor Recipes that I am not sure I need.

```python
if ENABLE_ALLAUTH:
    ENABLE_ALLAUTH_PROVIDERS = os.getenv('PAPERLESS_ENABLE_ALLAUTH_PROVIDERS').split(',') if os.getenv('PAPERLESS_ENABLE_ALLAUTH_PROVIDERS') else []
    INSTALLED_APPS = INSTALLED_APPS + ENABLE_ALLAUTH_PROVIDERS
    try:
        SOCIALACCOUNT_PROVIDERS = ast.literal_eval(
            os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') if os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') else '{}')
    except ValueError:
        SOCIALACCOUNT_PROVIDERS = json.loads(
            os.getenv('PAPERLESS_ALLAUTH_PROVIDERS').replace("'", '"') if os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') else '{}')
    AUTHENTICATION_BACKENDS = [
        "django.contrib.auth.backends.ModelBackend",
        "allauth.account.auth_backends.AuthenticationBackend",
    ]
```
This sets the necessary parameters if ENABLE_ALLAUTH is set to true.

```python
    # Points to custom account adapter to disable signup
    ACCOUNT_ADAPTER = "paperless.adapter.CustomAccountAdapter"

    # Variable to tell CustomAccountAdapter whether to allow signups
    ACCOUNT_ALLOW_SIGNUPS = False
```
This part was added to the above to make sure sign up was unavailable. Otherwise anyone could create a new account. See adapter.py for the remaining configuration.

The next part is required to verify the email address associated with the user. It is not necessary because you can login with your Paperless-ngx admin user

```python
# Outbound email configuration
EMAIL_HOST = os.getenv("PAPERLESS_EMAIL_HOST", "")
EMAIL_PORT = int(os.getenv("PAPERLESS_EMAIL_PORT", 25))
EMAIL_HOST_USER = os.getenv("PAPERLESS_EMAIL_HOST_USER", "")
EMAIL_HOST_PASSWORD = os.getenv("PAPERLESS_EMAIL_HOST_PASSWORD", "")
EMAIL_USE_TLS = __get_boolean("PAPERLESS_EMAIL_USE_TLS")
EMAIL_USE_SSL = __get_boolean("PAPERLESS_EMAIL_USE_SSL")
DEFAULT_FROM_EMAIL = os.getenv("PAPERLESS_DEFAULT_FROM_EMAIL", "webmaster@localhost")
ACCOUNT_EMAIL_SUBJECT_PREFIX = os.getenv("PAPERLESS_ACCOUNT_EMAIL_SUBJECT_PREFIX", "[Paperless] ") # Django Allauth sender prefix
```

### All changes to settings.py
```python
import ast

# Reverse Proxy requirements
REVERSE_PROXY_AUTH = __get_boolean("PAPERLESS_REVERSE_PROXY_AUTH")

if REVERSE_PROXY_AUTH:
    USE_X_FORWARDED_HOST = True
    SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")

# Enable Django Allauth
ENABLE_ALLAUTH = __get_boolean("PAPERLESS_ENABLE_ALLAUTH")

# Set Defualt access for Django Allauth
ALLAUTH_DEFAULT_ACCESS = __get_boolean("PAPERLESS_ALLAUTH_DEFAULT_ACCESS")
ALLAUTH_DEFAULT_GROUP = os.getenv('PAPERLESS_ALLAUTH_DEFAULT_GROUP', 'guest')

if ENABLE_ALLAUTH:
    ENABLE_ALLAUTH_PROVIDERS = os.getenv('PAPERLESS_ENABLE_ALLAUTH_PROVIDERS').split(',') if os.getenv('PAPERLESS_ENABLE_ALLAUTH_PROVIDERS') else []
    INSTALLED_APPS = INSTALLED_APPS + ENABLE_ALLAUTH_PROVIDERS
    try:
        SOCIALACCOUNT_PROVIDERS = ast.literal_eval(
            os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') if os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') else '{}')
    except ValueError:
        SOCIALACCOUNT_PROVIDERS = json.loads(
            os.getenv('PAPERLESS_ALLAUTH_PROVIDERS').replace("'", '"') if os.getenv('PAPERLESS_ALLAUTH_PROVIDERS') else '{}')
    AUTHENTICATION_BACKENDS = [
        "django.contrib.auth.backends.ModelBackend",
        "allauth.account.auth_backends.AuthenticationBackend",
    ]
    # Points to custom account adapter to disable signup
    ACCOUNT_ADAPTER = "paperless.adapter.CustomAccountAdapter"

    # Variable to tell CustomAccountAdapter whether to allow signups
    ACCOUNT_ALLOW_SIGNUPS = False

# Outbound email configuration
EMAIL_HOST = os.getenv("PAPERLESS_EMAIL_HOST", "")
EMAIL_PORT = int(os.getenv("PAPERLESS_EMAIL_PORT", 25))
EMAIL_HOST_USER = os.getenv("PAPERLESS_EMAIL_HOST_USER", "")
EMAIL_HOST_PASSWORD = os.getenv("PAPERLESS_EMAIL_HOST_PASSWORD", "")
EMAIL_USE_TLS = __get_boolean("PAPERLESS_EMAIL_USE_TLS")
EMAIL_USE_SSL = __get_boolean("PAPERLESS_EMAIL_USE_SSL")
DEFAULT_FROM_EMAIL = os.getenv("PAPERLESS_DEFAULT_FROM_EMAIL", "webmaster@localhost")
ACCOUNT_EMAIL_SUBJECT_PREFIX = os.getenv("PAPERLESS_ACCOUNT_EMAIL_SUBJECT_PREFIX", "[Paperless] ") # Django Allauth sender prefix
```

## adapter.py
I added an adapter.py file to ```src/paperless``` to stop people from creating new accounts.

```python
from django.conf import settings

from allauth.account.adapter import DefaultAccountAdapter

class CustomAccountAdapter(DefaultAccountAdapter):
    def is_open_for_signup(self, request):
        """
        Whether to allow sign ups.
        """
        allow_signups = super(
            CustomAccountAdapter, self).is_open_for_signup(request)
        # Override with setting, otherwise default to super.
        return getattr(settings, 'ACCOUNT_ALLOW_SIGNUPS', allow_signups)
```

## urls.py
This part breaks the pretty login form because it starts to use the default login forms that come with django-allauth. This was changed on ```src/paperless/urls.py```.
Change is as follows.

```diff
-path("accounts/", include("django.contrib.auth.urls")),
+path("accounts/", include("allauth.urls")),
```

## paperless.conf
All of the changes made here are matched up with the changes in settings.py. The only one that needs some explanation is the PAPERLESS_ENABLE_ALLAUTH_PROVIDERS and PAPERLESS_ALLAUTH_PROVIDERS which is explained by [Tandoor Recipes docs](https://docs.tandoor.dev/features/authentication).

```python
# Django Allauth settings
# If you are running Paperless behind a reverse proxy you will need to enable this
PAPERLESS_REVERSE_PROXY_AUTH=true
# Enable the use of Django Allauth
PAPERLESS_ENABLE_ALLAUTH=true
# Comma seperated list of Allauth providers https://django-allauth.readthedocs.io/en/latest/installation.html
PAPERLESS_ENABLE_ALLAUTH_PROVIDERS=allauth.socialaccount.providers.keycloak
PAPERLESS_ALLAUTH_PROVIDERS={"keycloak":{"KEYCLOAK_URL":"https://keycloak.domain.com","KEYCLOAK_REALM":"master"}}

# Outbound Mail
# Outbound email configuration
PAPERLESS_EMAIL_HOST=smtp.lxd
PAPERLESS_EMAIL_PORT=25
#PAPERLESS_EMAIL_HOST_USER=
#PAPERLESS_EMAIL_HOST_PASSWORD=
PAPERLESS_EMAIL_USE_TLS=false
PAPERLESS_EMAIL_USE_TLS=false
PAPERLESS_DEFAULT_FROM_EMAIL=paperless@domain.com
#PAPERLESS_ACCOUNT_EMAIL_SUBJECT_PREFIX=
```
## Django Admin Configuration
I then needed to do the configuration in django admin.
First you need to modify the Site to the FQDN that it will be working with. In the example I have used domain.com.

![](/assets/img/2022-08-03-adding-oauth-to-paperless/django-allauth-sites.png)

Then you need to go to the Social Application section and click on

![](/assets/img/2022-08-03-adding-oauth-to-paperless/django-allauth-add-socialapp.png)

Fill out the add social application as needed.

![](/assets/img/2022-08-03-adding-oauth-to-paperless/django-allauth-socialapp.png)

Next you need to create a user to link to your Keycloak Account. Once the user is created you can go to Social Account section and click on

![](/assets/img/2022-08-03-adding-oauth-to-paperless/django-allauth-add-socialaccount.png)

Fill out the add social application as needed.

![](/assets/img/2022-08-03-adding-oauth-to-paperless/django-allauth-socialaccount.png)

You should now be able to sign out and sign back in, by clicking on Keycloak.
