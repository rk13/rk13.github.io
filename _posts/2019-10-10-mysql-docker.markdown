---
layout: post
title:  "Mysql Docker"
date:   2019-10-10 12:00:00
---

### Prelude 
It was an early Saturday morning when I have received a message from one of my personal small project owners that site seems to be down. Although monitoring was not reporting any anomaly.

It was the beginning of a very long day.

### Investigation 
The affected site was running in a separate virtual environment and had its own MySQL database instance in the form of Docker container running on "mayer" (our primary working horse at Hetzner).

Site engine reported error connection to the database engine using a dedicated user, although the database container was running fine.
However, after connecting to the database as admin I haven't found any database schemas except for 'WARNING' with one table

```
mysql> show tables;
+-------------------+
| Tables_in_WARNING |
+-------------------+
| Readme            |
+-------------------+
1 row in set (0.00 sec)
```

```
mysql> select Description from Readme;
Your Database is downloaded and backed up on our secured servers. To recover ...
```

At this moment it became obvious that the database container was hacked and we should restore the database from backup, but before we should find out where the hole is ...

### Docker and UFW  
Some facts I knew at that time.
 
* Host machine was running Debian and utilized UFW (uncomplicated firewall) that is a very nice and simple frontend for iptables. 
* Docker container running site Mysql database was not intended to be accessible outside. However, the host was also running some containers intended to be accessible to the outside world (e.g. nginx).


UFW makes it possible to setup a firewall without having to fully understand iptables itself. Things do get complicated when you, however, are using Docker and you want to combine Docker with the UFW service.

As you probably know Docker uses a bridge to manage container networking. And in [Docker related documentation](https://docs.docker.com/v1.5/articles/networking/#the-world) it is defined whether a container can talk to the world by the ```ip_forward``` system parameter. Packets can only pass between containers if this parameter is 1. 

Usually, you will simply leave the Docker server at its default setting --ip-forward=true and Docker will go set ip_forward to 1 for you when the server starts up. "ip_forward" should be on, to at least make communication possible between containers and the wider world.

It should be noted that unless you specified ```--iptables=false``` when the daemon starts, Docker server will append forwarding rules to the DOCKER filter chain directly. And Docker's forward rules permit all external source IPs by default.
By default, UFW drops all forwarding traffic. But in our case default rules for UFW were the following ```DEFAULT_FORWARD_POLICY="ACCEPT"```.

### Solution

This was enough to understand that the problem was caused by site MySql container being open to the wider world and it was only a matter of time when the database password is bruteforced. 

Similar problem for MongoDB containers was discussed by [TechRepublic here](https://www.techrepublic.com/article/how-to-fix-the-docker-and-ufw-security-flaw/). 
The siggested fix was to go back to add ```DOCKER_OPTS="--iptables=false"```. 
However, in our case Docker was also running some containers intended to be accessible to the outside world, so not appending forwarding rules to DOCKER filter chain at all was not an option.

A quick solution was to identify all containers with exposed ports outside and restrict ip to server internal interface if the container was intended to be accessible from another container or to the public if it should be accessible to the outside world 
```
docker run -p $ip:$port:$mysql_port
```

Final simple script I used to run this and all other mysql containers

```
# internal ip / port
ip=
port=
db_name=
db_user=
db_password=

hostname=`basename $PWD`
db_dir="docker-volumes-mysql/$hostname"
init_dir="docker/$hostname/docker-entrypoint-initdb.d"

sudo docker stop $hostname
sudo docker rm -f $hostname

sudo docker run 
-p $ip:$port:3306 
--restart always 
-v "$db_dir":/var/lib/mysql 
-v "$init_dir":/docker-entrypoint-initdb.d 
--name $hostname 
-e MYSQL_DATABASE=$db_name 
-e MYSQL_USER=$db_user 
-e MYSQL_PASSWORD=$db_password 
-d mysql:latest
```

After that, it was pretty easy to restore container data from the backup, but this is a story for next time :).

See ya!





