---
layout: post
title: "Redirect all port 80 requests to port 8080"
date: 2012-04-23 09:50
comments: true
categories:
---

[In my previous post](http://echodittolabs.org/blog/2012/04/os-x-107-lion-development-nginx-php-mariadb-homebrew), I walked through how to set up a local environment using Nginx running on port 8080 so as to avoid running anything as root or with sudo. Something that I've found incredibly annoying is when I forget to specify the port I get an error in my browser, or Chrome might even suggest something based on a search term. It's fairly easy though to configure Apache to route everything to another port.

Create the file */etc/apache2/other/port8080-redirect.conf* as root. It's probably easiest to hop into Terminal and use nano:

```bash
sudo nano -w /etc/apache2/other/port8080-redirect.conf
```

Enter the following lines and save and exit the editor.

```apache
<VirtualHost _default_:80>
  DocumentRoot /Library/WebServer/Documents
  RewriteEngine On
  # Redirect all requests to the local Apache server to port 8080
  RewriteRule ^.*$ http://%{HTTP_HOST}:8080%{REQUEST_URI}
</VirtualHost>
```

Even though the *DocumentRoot* doesn't appear to be necessary, I wasn't able to get this working without it.

Go into System Preferences, open the Sharing preference pane, and check the box for Web Sharing. If you have trouble getting the check to "stick," you can run the following in Terminal: `sudo apachectl restart`

Now if you go in your browser and hit a domain you have defined in */etc/hosts*, like **http://drupal.local/**, you'll automatically be redirected to **http://drupal.local:8080/**