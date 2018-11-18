---
layout: post
title:  "Mysql Docker"
date:   2018-10-10 12:00:00
---

It was an early Saturday morning when I have received a message from one of my personal small project owners that site seems to be down. Although Hetzner monitoring was not reporting any anomaly... It was the beginning of a very long day.

The site was running in a separate virtual environment and has its own MySQL database instance in the form of Docker container running on "mayer" (our primary working horse at Hetzner).

Site engine reported error connection to the database engine using a dedicated user, although the database container was running fine.

After connecting to database admin I haven't found any database schemas except for 'WARNING' with one table

```
mysql> show tables;
+-------------------+
| Tables_in_WARNING |
+-------------------+
| Readme            |
+-------------------+
1 row in set (0.00 sec)

mysql> select Description from Readme;
Your Database is downloaded and backed up on our secured servers. To recover your lost data: Send 0.6 BTC to our BitCoin Address and Contact us by eMail with your server IP Address and a Proof of Payment. Any eMail without your server IP Address and a Proof of Payment together will be ignored. We will drop the backup after 24 hours. You are welcome!.
```

docker -> --ip-forward=true
sudo docker run  -p $ip:$port:3306

