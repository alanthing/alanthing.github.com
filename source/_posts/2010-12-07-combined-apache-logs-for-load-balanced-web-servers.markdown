---
layout: post
title: "Combined Apache logs for load balanced web servers"
date: 2010-12-07 14:34
comments: true
categories: 
---

When dealing with a load spike or unexplained poor performance, once of the first things to do is to check the Apache web server logs. However, when dealing with multiple web servers behind a load balancer, it can be cumbersome to check through multiple logs on different machines.

Luckily, if you have shared storage, it’s pretty simple to write to a single log file. Apache allows for log file locations to go to a pipe, which means you can redirect the output anywhere else. This was initially done to allow Apache to release open file descriptors since it can begin to create problems with more than 300 domains. But you don’t need to have 300 websites to see the benefits of using a pipe for logging.

In your VirtualHost declaration, change your logging options from:

	CustomLog logs/domain.com-access_log combined

to:

	CustomLog "|/bin/cat » /var/www/vhosts/_logs/domain.com-access_log" combined

In this example, /var/www/vhosts is the shared SAN storage device, and by using “cat »” we append Apache output in real-time to the log file. As multiple machines do this simultaneously, you begin to have a single log file from multiple machines. You can do the same thing with your ErrorLog:

	ErrorLog  "|/bin/cat » /var/www/vhosts/_logs/domain.com-error_log"

There is a slight problem with this approach: it doesn’t give us enough information. If one of the 4 servers behind the load balancer was the source of a problem, it would be helpful to determine which server each log entry is coming from. 

For CustomLog, we can create a new LogType that adds the server’s IP address at the end of the line, and the LogType name will be “combinedIP”:

	LogFormat "%h %l %u %t "%r" %>s %b "%{Referer}i" "%{User-Agent}i" %A" combinedIP

This format is identical to “combined” and just adds %A at the end. Then, simply change the end of the CustomLog line:

	CustomLog "|/bin/cat » /var/www/vhosts/_logs/domain.com-access_log" combinedIP

Here’s an example of what your log file will look like, showing entries from two servers:

	1.2.3.4 - - [07/Dec/2010:11:22:08 -0500] "GET /blog/feed HTTP/1.1" 304 - "-" "-" 192.168.1.6
	5.6.7.8 - - [07/Dec/2010:11:22:08 -0500] "GET /blog/feed HTTP/1.1" 304 - "-" "-" 192.168.1.8

These log entries are the same as the typical “combined” LogType but with the addition of the IP address at the end.

But what about the ErrorLog? There is no way to define a LogType for the ErrorLog, and it would be helpful to know which server originated error, say, if MaxClients was reached, or there is a misconfiguration in a configuration file. Since we’re piping the ErrorLog output to /bin/cat, we can instead pipe it to a script that simply adds the server hostname on the end (I chose not to use an IP address because 1) some servers have multiple IP addresses, and 2) I cannot insert the VirtualHost’s IP address automatically. Better to know the server and then determine the IP address is necessary). We’ll start by creating a simple script at /usr/local/bin/httpd_errors (I’m not using bash since I was not able to get it to write to the log file in real time, it would only write once Apache was reloaded. I did not spend much time to determine why, and I knew it would work in PHP):

```php
#!/usr/bin/php -q
<?php
$stdin = fopen (‘php://stdin’, ‘r’);
ob_implicit_flush (true); // Use unbuffered output
while ($line = fgets ($stdin))
{
    print chop($line).” - “.system(‘hostname -f’).”\n”;
}
?>
```

Make the script executable with “chmod +x /usr/local/bin/httpd_errors”, and then change your ErrorLog in your Apache configuration:

	ErrorLog  ”|/usr/local/bin/httpd_errors » /var/www/vhosts/_logs/domain.com-error_log”

And your error logs will now reveal which server the error originated from:

	[Tue Dec 07 11:20:51 2010] [error] [client 1.2.3.4] File does not exist: /var/www/vhosts/domain.com/favicon.ico - wsrv1
	[Tue Dec 07 11:24:27 2010] [error] [client 5.6.7.8] File does not exist: /var/www/vhosts/domain.com/favicon.ico - wsrv2

There you have it, combined Apache logs for multiple web servers. If you have any suggestions, questions, tips, etc, leave a comment!