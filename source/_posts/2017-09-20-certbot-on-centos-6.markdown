---
layout: post
title: "Certbot on CentOS 6"
date: 2017-09-20 20:32
comments: true
categories: 
---

CentOS 6 is getting updates through [November 30, 2020](https://wiki.centos.org/FAQ/General#head-fe8a0be91ee3e7dea812e8694491e1dde5b75e6d), but it's getting more and more difficult to find newer packages for the operating system. If you want to use [Certbot](https://certbot.eff.org) for obtaining and renewing [Let's Encrypt](https://letsencrypt.org) TLS certificates, you can use [certbot-auto](https://certbot.eff.org/#centos6-other) and let it handle the work for you, but I wanted to try only [the PyPi package](https://pypi.python.org/pypi/certbot/) via pip.

*Note:* I'm using Docker to test these commands via: `docker run --rm -it centos:6 bash`

```
# No Python wheels available, so we need gcc and some devel packages
$ yum install gcc libffi-devel openssl-devel python-devel python-pip

# Let's try and install certbot
$ pip install --upgrade certbot
...snip...
Successfully installed ConfigArgParse-0.12.0 PyOpenSSL-17.3.0 acme-0.18.2 certbot-0.18.2 cffi-1.11.0 configobj-5.0.6 cryptography-2.0.3 funcsigs-1.0.2 mock-2.0.0 pbr-3.1.1 pyrfc3339-1.0 requests-2.18.4 zope.component-4.4.0 zope.event-4.3.0
```

Seemed to work, right? Nope 😠:

```
$ certbot --help
Traceback (most recent call last):
  File "/usr/bin/certbot", line 7, in <module>
    from certbot.main import main
  File "/usr/lib/python2.6/site-packages/certbot/main.py", line 7, in <module>
    import zope.component
  File "/usr/lib/python2.6/site-packages/zope/component/__init__.py", line 28, in <module>
    from zope.component.globalregistry import getGlobalSiteManager
  File "/usr/lib/python2.6/site-packages/zope/component/globalregistry.py", line 18, in <module>
    from zope.interface.registry import Components
  File "/usr/lib64/python2.6/site-packages/zope/interface/registry.py", line 167
    filtered_state = {k: v for k, v in reduction[2].items()
                             ^
SyntaxError: invalid syntax
```

[Of course this doesn't work](https://stackoverflow.com/q/345356/534275), it's Python **2.6**!

[Software Collections](https://wiki.centos.org/AdditionalResources/Repositories/SCL), to the rescue! Let's use the Python 2.7 SCL and try again:

```
# Install the SCL Yum repo files
$ yum install centos-release-scl

# Grab Python 2.7, pip, and related depedencies
$ yum install python27-python-pip

# Install into the SCL root with pip. Wheels are available, so no compiler or devel packages necessary!
$ scl enable python27 'pip install certbot'
...snip...
Successfully installed ConfigArgParse-0.12.0 PyOpenSSL-17.3.0 acme-0.18.2 asn1crypto-0.22.0 certbot-0.18.2 certifi-2017.7.27.1 cffi-1.11.0 chardet-3.0.4 configobj-5.0.6 cryptography-2.0.3 enum34-1.1.6 funcsigs-1.0.2 future-0.16.0 idna-2.6 ipaddress-1.0.18 mock-2.0.0 parsedatetime-2.4 pbr-3.1.1 pycparser-2.18 pyrfc3339-1.0 pytz-2017.2 requests-2.18.4 setuptools-36.5.0 six-1.11.0 urllib3-1.22 zope.component-4.4.0 zope.event-4.3.0 zope.interface-4.4.2
```

Success! Right?

```
$ certbot --help
bash: certbot: command not found
```

Hrm, what happened?

```
$ find /opt/rh/python27/ -name certbot -executable
/opt/rh/python27/root/usr/bin/certbot
/opt/rh/python27/root/usr/lib/python2.7/site-packages/certbot
```

Right, it's in the SCL. You have some options. You could [enable the SCL in userspace](https://access.redhat.com/solutions/527703) through commands like `source scl_source enable python27` or `scl enable python27 bash`. Or, because I want to use `certbot` interactively and in crons without thinking about the SCL, we can use a "shim":

```
$ cat > /usr/local/bin/certbot <<'EOF'
#!/bin/bash
source scl_source enable python27
certbot "$@"
EOF
$ chmod -c 755 /usr/local/bin/certbot
```

Assuming `/usr/local/bin/` is in your `$PATH`, we can test this:

```
$ certbot register --agree-tos -m 'le@example.com' -n
Saving debug log to /var/log/letsencrypt/letsencrypt.log

IMPORTANT NOTES:
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.

$ find /etc/letsencrypt \! -type d
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/a1b2c3d4e5f6a7b8a1b2c3d4e5f6a7b8/meta.json
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/a1bx2c3d4e5f6a7b8a1b2c3d4e5f6a7b8/regr.json
/etc/letsencrypt/accounts/acme-v01.api.letsencrypt.org/directory/a1b2c3d4e5f6a7b8a1b2c3d4e5f6a7b8/private_key.json
```

Now you have `certbot` on CentOS 6 from PyPi and without `certbot-auto` (Nothing wrong with `certbot-auto`, I just prefer to have more control over the installation process). Let me know how it goes for you!
