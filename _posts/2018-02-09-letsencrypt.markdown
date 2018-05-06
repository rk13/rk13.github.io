---
layout: post
title:  "HTTPS with Let's Encrypt"
date:   2018-02-08 12:00:00
---

As you might know, modern browsers mark HTTP sites in the address bar as unsafe. If you have one domain, it's usually not a problem to buy a certificate for it directly from your hosting provider. But if there are many sites or one site, but with a large number of subdomains, the price of certificates or one wildcard certificate will just be unreasonable. The Let's Encrypt certification center, available at the expense of sponsors and issuing certificates for free, comes to the aid.

In addition to free, Let's Encrypt is interesting because it issues certificates automatically, with the help of a special client - Certbot. The certificate is valid for 90 days and is also renewed automatically.

As we have many sites running on curie without HTTPS it was the time to fix this :).


#### Steps to install Certbot:

```
sudo add-apt-repository ppa: certbot / certbot
sudo apt-get update
sudo apt-get install python3-certbot-nginx
```

In the nginx configuration, we check that the server_name parameter explicitly specifies the domain name of your site, or several names through a space.

```
# static distribution
server {
    listen 80 default_server;

    charset UTF-8;

    root /home/site-www/site.lv;
    index index.html index.htm;

    server_name site.lv;

    location / {
        try_files $ uri $ uri / = 404;
    }
}

# reverse proxy
server {
    listen 80 default_server;
    listen [::]: 80 default_server ipv6only = on;

    server_name site.lv;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host site.lv;
        proxy_set_header X-Real-IP $ remote_addr;
        proxy_set_header X-Forwarded-Proto $ scheme;
        proxy_set_header X-Forwarded-For $ proxy_add_x_forwarded_for;
    }
}
```

```
sudo service nginx reload
sudo certbot --nginx -d site.lv
```

Thanks to https://eax.me/lets-encrypt/

