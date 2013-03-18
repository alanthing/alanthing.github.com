---
layout: post
title: "Dump each MySQL table to a file"
date: 2011-02-22 14:30
comments: true
categories:
---

Here's a one-liner to dump each table in a database to it's own .sql file. Crack open your shell of choice and follow along.

Replace the USER, PASSWORD, and DBNAME values with your own. If you're not running this to connect to a local database, add --host=domain.tld after each password to connect to your remote server.

```
for i in $(mysql -uUSER -p'PASSWORD' --batch --skip-column-names DBNAME -e'show tables;'); do mysqldump -uUSER -p'PASSWORD' DBNAME $i > DBNAME-$i.sql; done
```

Users, passwords, and database names are case sensitive, I made them uppercase here to call them out. Leave a comment if you have a question or notice a problem or typo.