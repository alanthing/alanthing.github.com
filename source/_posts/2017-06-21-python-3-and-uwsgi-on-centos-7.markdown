---
layout: post
title: "Python 3 and uWSGI on CentOS 7"
date: 2017-06-21 07:04
comments: true
categories: 
---

Wherever I can, I avoid compiling applications from source or packaging up my own versions of things that exist in popular repositories. I recently needed to run a Python 3 application via uWSGI on CentOS 7 and was frustrated at how many search results were recommending building uWSGI from source when it's [available in EPEL](https://dl.fedoraproject.org/pub/epel/7/x86_64/u/). I've only deployed a handful of Python applications, and was not interested in keeping up with source-compiled dependencies to support this small project over time. We'll discuss a couple of different ways to run these programs together with minimal customization.

# Python 3.4

In CentOS 7, you can run Python 3.4 alongside Python 2.7 as it is available in [EPEL](https://dl.fedoraproject.org/pub/epel/7/x86_64/p/).

```bash
rpm -q epel-release || yum -q -y install epel-release

yum -y install python34
```

## pip

If you want to install [pip](https://pip.pypa.io), simply `yum install python34-pip` and it will install at `/usr/bin/pip3` and `/usr/bin/pip3.4`, leaving `/usr/bin/pip` alone as that is part of the `python2-pip` package. This was only available recently (late 2016); previously you had to manually install and deal with `/usr/bin/pip` being overwritten.

## Virtual Environments

Since 3.3, Python supports creating virtual environments without an external module like `virtualenv`. `pyvenv` was the "recommended tool" for this initially, though it [is deprecated in Python 3.6](https://docs.python.org/dev/whatsnew/3.6.html#id8). As such, to lean into the preferred Python nomenclature, you can use the [venv module](https://docs.python.org/3/library/venv.html) directly to create Python 3.4 virtual environments:

```bash
python3.4 -m venv /path/to/venv34dirname

source /path/to/venv34dirname/bin/activate
```

If you like using [virtualenv](https://virtualenv.pypa.io), you'll be disappointed to see that there is no `python34-virtualenv` package available in EPEL. While you could `pip3 install virtualenv`, that would install at `/usr/bin/virtualenv`, potentially overwriting the same file provided by the `python-virtualenv` package. But, if you do, consider the following:

```bash
# Install virtualenv with the pip provided the 'python34' package:
pip3 install virtualenv

# Then, a virtualenv based on /usr/bin/python3.4:
virtualenv /path/to/venv34dirname

# Now, to create based on /usr/bin/python2.7, using a newer version of virtualenv than is provided by the 'python-virtualenv' package:
virtualenv --python=/usr/bin/python2.7 /path/to/venv27dirname

# Or, recommended, use the alternate path from the 'python-virtualenv' package:
virtualenv-2.7 /path/to/venv27dirname
```

Unless you have a compelling reason for `virtualenv` on your servers, use the built-in `venv` module. I don't like to overwrite files provided by packages, and we should use recommended upstream tools where possible.

## uWSGI

Since uWSGI is provided by EPEL, there is no need to install it with `pip`.

```bash
yum -y install uwsgi uwsgi-plugin-python3
```

This will install the Python 3.4 uWSGI plugin to `/usr/lib64/uwsgi/python3_plugin.so`. Be sure to include the directive `plugin = python3` in your uWSGI definition and proceed as normal (beyond the scope of this post).

# Python 3.5 Software Collection

I love [Software Collections](http://softwarecollections.org). With Python 2.7 being the default version of Python with CentOS 7, I find it much better to keep another version in a completely separate directory structure than integrating with `/usr` and such. We'll use a rebuild of the Red Hat SCL for Python 3.5, and everything will be installed to `/opt/rh/rh-python35`.

```bash
yum -y install centos-release-scl

yum -y install rh-python35-python
```

## Using

You'll quickly notice that the newly-installed Python 3.5 binary is not in your `$PATH`; it's in `/opt/rh/rh-python35/root/bin`. You have a few options:

Launch a new instance of bash:

```bash
scl enable rh-python35 bash
```

Alternatively, you can modify your existing session by running one of the following manually, or add to `~/.bashrc`:

```bash
# Preferred method:
source scl_source enable rh-python35

# Source the file directly:
#source /opt/rh/rh-python35/enable
```

## pip

```bash
yum -y install rh-python35-python-pip
```

After installing, you can use `pip`, `pip3`, or `pip3.5`

## Virtual Environments

You can use the built-in method:

```bash
python3.5 -m venv /path/to/venv35dirname
```

Or, you can use `virtualenv` (though, see the 3.4 section for the rationale I recommend against this):

```bash
yum -y install rh-python35-python-virtualenv

virtualenv-3.5 /path/to/venv35dirname
```

## uWSGI

While there is a `uwsgi-plugin-python3` RPM, it's for the aforementioned `python34` package. This is where you will have to compile something manually; so carefully consider the pros and cons of using a Software Collection. _(Later, I may add to this post where you can use a yum post-install or post-upgrade action to ensure the compiled plugin is kept up to date.)_

This is a helpful exercise to learn how you can integrate packages that install to default paths and a Software Collection which is kept away from default paths.

```bash
yum -y install uwsgi uwsgi-plugin-common

UWSGI_VERSION="$( rpm -q uwsgi --queryformat '%{VERSION}' )"
UWSGI_PLUGINS_DIR="$( dirname "$( rpm -q uwsgi-plugin-common -l | grep '_plugin\.so' | tail -1 )" )"

source scl_source enable rh-python35

if [[ $X_SCLS == *rh-python35* ]]; then
  UWSGI_PLUGINS_DIR="/opt/rh/rh-python35/root${UWSGI_PLUGINS_DIR:?}"
fi

if [[ ! -f "${UWSGI_PLUGINS_DIR:?}/python35_plugin.so" ]]; then
  yum -y install gcc libcap-devel libuuid-devel make openssl-devel rh-python35-python-devel

  mkdir -pv /opt/rh/rh-python35/root/src
  cd /opt/rh/rh-python35/root/src

  curl -LO "https://projects.unbit.it/downloads/uwsgi-${UWSGI_VERSION:?}.tar.gz"
  tar zxvf "uwsgi-${UWSGI_VERSION:?}.tar.gz"
  cd "uwsgi-${UWSGI_VERSION:?}/"

  make PROFILE=nolang
  PYTHON=python3.5 /usr/sbin/uwsgi --build-plugin "plugins/python python35"

  [[ ! -d "${UWSGI_PLUGINS_DIR:?}/" ]] && mkdir -pv "${UWSGI_PLUGINS_DIR:?}/"
  mv -v python35_plugin.so "${UWSGI_PLUGINS_DIR:?}/"
fi
```

_(Reference: [http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html](http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html))_

After completing, your resulting uWSGI file will need to contain:
```ini
plugins-dir = /opt/rh/rh-python35/root/usr/lib64/uwsgi
plugin = python35
```

# Which should you use?

If you need Python 3 but want to reduce the amount of customization that you need to do, I recommend installing `python34` from EPEL. You can use it with uWSGI with no compiling from source.

If Python 3.4 is too old and/or you need 3.5+ functionality, the Software Collection can be an alternative to compiling Python from source and knowing that you're getting quality packages designed by Red Hat and rebuilt by CentOS.

If you need Python 3.6, or a newer release of 3.4 or 3.5 not provided quickly enough by the package maintainers, and you don't want to deal with all of the pain of `./configure && make && sudo make install`, I would suggest looking at [pyenv](https://github.com/pyenv/pyenv) to meet your needs.

Please let me know if I missed something or should add anything. I'm cumulatively green with running Python in production but wanted to share my learnings as I was frustrated by the lack of good information for my needs. Feedback is welcome.