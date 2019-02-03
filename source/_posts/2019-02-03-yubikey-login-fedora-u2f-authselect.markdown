---
layout: post
title: "Yubikey login to Fedora with U2F via Authselect"
date: 2019-02-03 16:54
comments: true
categories: 
---

If you have a [Yubikey with U2F support for Linux](https://support.yubico.com/support/solutions/articles/15000011356-ubuntu-linux-login-guide-u2f#Applicable_Productsb6c8j5), you can use its U2F functionality for a 2nd factor or single factor for logins, sudo passwords, and more. This is accomplished by the [pam-u2f](https://developers.yubico.com/pam-u2f/) module, and the instructions commonly returned in searches aren't for the faint of heart, especially when get into editing the files in _/etc/pam.d/_. Recent versions of [Fedora ship with `authselect`](https://fedoraproject.org/wiki/Changes/Authselect) which can make this process a lot easier. U2F support isn't baked in,s but it's easy enough to find on GitHub. Grab your Yubikey<u>s</u> _(you do have a primary and a backup, right?)_ and follow along.

The main packages required are `authselect`, `pam-u2f`, and `pamu2fcfg`. The following will cover the bases in case you have a minimal install:

```bash
dnf install authselect bash gawk pam pam-u2f pamu2fcfg sssd systemd-udev
```

Create the _u2f\_keys_ file and include values from as many U2F devices as applicable ([additional background here](https://wiki.gentoo.org/wiki/Pam_u2f#Registration)):

```bash
# pam-u2f looks in this directory
mkdir -pv ~/.config/Yubico/

pamu2fcfg --username="$USER" | tee ~/.config/Yubico/u2f_keys ; echo
# touch device

pamu2fcfg --nouser | tee -a ~/.config/Yubico/u2f_keys ; echo
# touch 2nd device

# Repeat the previous command as necessary
```

Grab a udev rules file to allow access for non-root users ([additional background here](https://support.yubico.com/support/solutions/articles/15000006449-using-your-u2f-yubikey-with-linux)):

```bash
curl \
  --output '/etc/udev/rules.d/70-u2f.rules' \
  'https://raw.githubusercontent.com/Yubico/libu2f-host/master/70-u2f.rules'

# Take effect without reboot:
udevadm trigger
```

With `pam-u2f` installed and the keys added in _u2f\_keys_, the final step is adding configurations to files in _/etc/pam.d/_; thankfully [authselect](https://github.com/pbrezina/authselect) makes this less intimidating. The version that ships with Fedora 29 (as of this writing) does not include the U2F option, so we'll create a custom profile based on _sssd_ and grab the files from GitHub. Uncomment the `diff` command to see how minimal the changes are that we are getting:

```bash
# Create a profile based on the built-in "sssd" and use symlinks for the files we won't
# be changing
authselect \
  create-profile \
  'sssd-u2f' \
  --base-on=sssd \
  --symlink-nsswitch \
  --symlink-dconf \
  --symlink=fingerprint-auth \
  --symlink=postlogin \
  --symlink=smartcard-auth

# Grab 4 updated files from GitHub. Uncomment `diff` to see the changes from the "sssd"
# profile
for FILE in README REQUIREMENTS password-auth system-auth; do
  curl \
    --silent \
    --output "/etc/authselect/custom/sssd-u2f/$FILE" \
    "https://raw.githubusercontent.com/pbrezina/authselect/297f48d/profiles/sssd/$FILE"

  # diff \
  #   --ignore-all-space \
  #   --unified \
  #   "/usr/share/authselect/default/sssd/$$FILE" \
  #   "/etc/authselect/custom/sssd-u2f/$$FILE"
done

# Capture the options your current authselect profile might be using. My system had
# "with-fingerprint" and "with-silent-lastlog" after a standard Fedora 29 install
mapfile -t AUTHSELECT_CURRENT_OPTIONS < <( authselect current | awk '/^- with-/ {gsub("^- ", ""); print $0}' )

# If you want, show the captured options, which might be blank
#printf '%s\n' "${AUTHSELECT_CURRENT_OPTIONS[@]}"

# Select the custom "sssd-u2f" profile and add the "with-pam-u2f" option
authselect \
  select \
  'custom/sssd-u2f' \
  $( printf '%s ' "${AUTHSELECT_CURRENT_OPTIONS[@]}" ) \
  'with-pam-u2f'
```

On my system, I was able to use my Yubikey's U2F mode to login after a reboot with Gnome, and for issuing `sudo`. To make the U2F required as part of a two-factor login or similar, you'll need to dig into the _/etc/pam.d/_ files. Rather than edit directly, edit the files in _/etc/authselect/custom/sssd-u2f/_ and apply with `authselect`.

For some further reading, the [RHEL 8 Beta page on authselect](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8-beta/html/installing_identity_management_and_access_control/using-authselect) has a ton of great information. I'm glad to see `authselect` will make it from Fedora to RHEL!

_(I haven't used Fedora 29 as a desktop for long, so please let me know [@alanthing](https://twitter.com/alanthing) if you have any feedback!)_
