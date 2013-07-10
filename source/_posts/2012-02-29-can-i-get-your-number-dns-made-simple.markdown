---
layout: post
title: "Can I get your number? DNS made simple"
date: 2012-02-29 09:51
comments: true
categories:
---

At [EchoDitto](http://www.echoditto.com), we get a lot of questions about how hosting works, and in particular about changing DNS records for new websites or websites who are moving to new hosting providers. Here's an old but true analogy to make understanding DNS easier: the IP address of your website is like your phone number (it changes if you move houses - at least, your landline does), and DNS is like the phonebook. If someone wants to call you at your new house they might look you up by name in the phonebook, but if their phonebook hasn't been updated since you moved they're going to call your old house.

Here's how it works in more detail:

When visitors want to access your website their computers take the domain name that they type in (echoditto.com, for example) and look up the IP address for that domain by way of Domain Name Service, or DNS.

To get your DNS records for users to get to your website, first you need a domain name. Go Daddy and Network Solutions are examples of registrars where you can buy domain names.

Continuing the analogy, the first thing a visitor's computer needs to know is which "phonebook" your website's IP address is listed in. The registrar will set something called the "name server" for your domain, which is akin to telling the user which "phonebook" (name server) to look you up in. The name server contains the details about how your domain name, example.com, will get to the computer that actually contains (or "hosts") your website.

In a phonebook, you look up a name and get a phone number. It's similar with DNS - the user's computer will look up your domain in the name server and get an IP address, which is a numerical address of a computer on the internet. The name server gives the user's computer the exact IP address of the computer that hosts your website. Once it has that information the user's computer can "call" the number,  which takes them directly to your website.

DNS contains another field along with the IP address, called a TTL, or time-to-live. This time, in seconds, is how long your computer should remember the number in the phonebook before it looks it up again. Computers check in with DNS name servers before visiting a site again to make sure the IP address of a website hasn't changed, and the TTL tells them how often to do that. TTL times can vary, but often you may find it initially set for 1 to 4 hours on average websites. The higher the setting is, the less often the DNS name server will be asked about the IP address and other DNS records.

When launching a website on a new server, or moving web hosts, you want to temporarily change the TTL to be really low. That makes website visitors check their "phonebook" more frequently, which means they will stop going to the old IP address as soon as possible after a change is made. It can take a while for a new TTL to take effect for all users if using the typical defaults of 1 to 4 hours, so when we're launching a new site, or changing hosting, we do this ahead of time. Once visitors worldwide have the new lower TTL, we can change the DNS entry for the website to point to the new IP address, and then visitors will see your new website, or your current website on the new host, very quickly.

Have other questions about DNS? Ask your computer to look up [http://www.youtube.com/watch?v=oN7ripK5uGM](http://www.youtube.com/watch?v=oN7ripK5uGM) to watch a video that explains all of this in another way.
