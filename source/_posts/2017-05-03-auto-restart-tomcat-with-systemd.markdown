---
layout: post
title: "Auto-restart Tomcat with systemd"
date: 2017-05-03 08:14
comments: true
categories: 
---

It's commonplace that if you've run services on Linux and wanted something to auto-restart if it crashed, then you've looked at [monit](https://mmonit.com/monit/). You may not know, however, that [systemd](https://www.freedesktop.org/wiki/Software/systemd/) can provide the ability to restart a failed process without adding another daemon. In this example, we'll keep Tomcat running if it stops for any reason, unless we `systemctl stop tomcat` using a systemd override:

```bash
mkdir -pv /etc/systemd/system/tomcat.service.d/

cat > /etc/systemd/system/tomcat.service.d/restart.conf <<'EOF'
[Service]
Restart=always
RestartSec=30
TimeoutStartSec=240
TimeoutStopSec=240
EOF

systemctl daemon-reload
```

You don't need to restart Tomcat; systemd will begin to honor the `Restart` setting immediately. You can view all of these values before and after to verify the changes were honored:

```bash
$ systemctl show tomcat | grep -E '^(Restart(|USec)|Timeout(Start|Stop)USec)='
Restart=always
RestartUSec=30s
TimeoutStartUSec=4min
TimeoutStopUSec=4min
```

I recommend reading the [documentation](https://www.freedesktop.org/software/systemd/man/systemd.service.html) about each of the options, but a quick explanation:

* `Restart=always`: Unless running `systemctl stop`, systemd will always attempt to start the process if it dies for any reason. That means if you were to `pkill -f tomcat` or any other kill command, regardless of the SIG code, it will be restarted. This would mean you should always be using `systemctl` to stop. If you'd like the ability to stop the process another way and only want systemd to restart if the application crashed or exited with a non-zero exit code, you can use `Restart=on-failure` or another value. Check the docs for your use-case.
* `RestartSec=30`: First, the documentation above lists `RestartSec` despite `systemctl show` returning `RestartUSec`; I try and stick with the docs where I can so you can follow along with the official guidance rather than something I have stumbled upon that may not work in the future. This value is how long to sleep before restarting the service. The default is only `100ms`, so I like to set it higher to allow for a busy operation to continue before systemd gets too aggressive with restarting.
* `TimeoutStartSec` and `TimeoutStopSec` instruct systemd how long to wait for the stop and start commands before considering the operation failed and trying again. Change as needed, but I set it to 4 minutes to allow for our Tomcat applications to completely initialize.

It's worth noting that you can run the interactive command `systemctl edit tomcat` and it will open `/etc/systemd/system/tomcat.service.d/override.conf` in your `$EDITOR` for adding in the contents above, eliminating the need to remember the directory structure and where to put override directives. I prefer adding files by hand as shown above for two reasons: 1) adding a file directly is obviously easier in a non-interactive server initialization script, and 2) you can have multiple files with the pattern `/etc/systemd/system/name.service.d/*.conf`. For organizing purposes, you could have a `restart.conf` where you specify your custom Restarting parameters, and another `environment.conf` file where you might add more environmental variables.