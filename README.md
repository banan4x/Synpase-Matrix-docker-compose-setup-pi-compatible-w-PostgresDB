# Tutorial: Synpase Matrix docker-compose setup (pi compatible) w/ PostgresDB

This tutorial describes the setup of a matrix homeserver hidden behind a VPN (so you don't have to expose your local IP and have all the data locally stored)

## Requirements:
- Small VPS with public IP
- any domain
- raspberry pi (I'm using the 4 1GB version currently)

## VPS Setup (using Debian 10 in this tutorial):
Setup your VPS as usual - setup nginx, certbot, ufw, fail2ban, stuff like that.
```
sudo apt install nginx, certbot, ufw, fail2ban
```
### Firewall setup (on the VPS)
Port 8448 is the matrix federation port.
Port 51194 is used by wireguard - if you've used a different port in the tuturial you'll also have to use it here
```
sudo systemctl stop nginx
sudo ufw allow http
sudo ufw allow https
sudo ufw allow ssh
sudo ufw allow 8448/tcp
sudo ufw allow 51194/udp
sudo ufw enable
sudo ufw status verbose
```
### Wireguard Setup (on the VPS)
And just follow this tutorial to setup the VPN/wireguard: https://www.cyberciti.biz/faq/debian-10-set-up-wireguard-vpn-server/

Once you've setup the VPN connection between the VPS and your Pi, verify it by pinging the other host:
(If you've used the same ip's as in the tutorial)
To check the connection from the pi:
```user@pi$ ping 192.168.10.1```

To check the connection from the vps:
```user@vps$ ping 192.168.10.2```

### Reverse Proxy Setup (on the VPS)
Make sure you've setup your domains DNS A Entry to point to your VPSs public IP.
Obtain a Let's Encrypt certificate using the certbot - for this nginx has to be turned off and the firewall has to be setup properly.
```
certbot certonly
```
```
cd /etc/nginx/sites-available
nano <yourdomain>.matrix.proxy.conf
```
#### <yourdomain>.matrix.proxy.conf
```
# HTTP redirect
server {
    listen      80;
    listen      [::]:80;
    server_name <yourdomain>;
    return      301 https://<yourdomain>$request_uri;
}

server {
    listen      443 ssl http2;
    listen      [::]:443 ssl http2;
    server_name <yourdomain>;

    listen 8448 ssl http2 default_server;
    listen [::]:8448 ssl http2 default_server;

    # SSL
    ssl_certificate     /etc/letsencrypt/live/<yourdomain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<yourdomain>/privkey.pem;

    #access_log          /var/log/nginx/<yourdomain>.access.log anonymized;
    error_log           /var/log/nginx/<yourdomain>.error.log warn;

    # reverse proxy
    location ~ ^(/_matrix|/_synapse/client) {
        
        #IP of your client in the VPN
        proxy_pass http://192.168.10.2:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;

        # Nginx by default only allows file uploads up to 1M in size
        # Increase client_max_body_size to match max_upload_size defined in homeserver.yaml
        client_max_body_size 350M;
    }
    gzip            on;
    gzip_vary       on;
    gzip_proxied    any;
    gzip_comp_level 6;
    gzip_types      text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
}
```

## Synapse/Matrix Setup
