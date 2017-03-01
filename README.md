# FusionPBX Debian 8 Install

    apt-get update && apt-get upgrade -y --force-yes
    apt-get install -y --force-yes git
    cd /usr/src
    git clone https://github.com/fusionpbx/fusionpbx-install.sh.git
    chmod 755 -R /usr/src/fusionpbx-install.sh
    cd /usr/src/fusionpbx-install.sh/debian
    ./install.sh

## After install, use the generated username/password to login to web UI, go to the navigation and do the following:

    Advanced -> Upgrade select the checkbox for App defaults then execute.
    Go to Status -> SIP Status and start the SIP profiles
    Go to Advanced -> Modules and find the module Memcached and click start.

# Lets Encrypt Install for NGINX

    cd /usr/src/
    git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
    cd /opt/letsencrypt
    chmod a+x ./certbot-auto
    cd /etc/letsencrypt/
    mkdir -p configs
    cd configs
    nano /etc/letsencrypt/configs/example.com.conf

## Put this into the .conf file, (edit the defaults):

    # the domain we want to get the cert for;
    # technically it's possible to have multiple of this lines, but it only worked
    # with one domain for me, another one only got one cert, so I would recommend
    # separate config files per domain.
    domains = my-domain

    # increase key size
    rsa-key-size = 2048 # Or 4096

    # the current closed beta (as of 2015-Nov-07) is using this server
    server = https://acme-v01.api.letsencrypt.org/directory

    # this address will receive renewal reminders
    email = my-email

    # turn off the ncurses UI, we want this to be run as a cronjob
    text = True

    # authenticate by placing a file in the webroot (under .well-known/acme-challenge/)
    # and then letting LE fetch it
    authenticator = webroot
    webroot-path = /var/www/letsencrypt/

## Then ...

    nano  /etc/nginx/sites-available/fusionpbx

## Add this after the ssl_ciphers line:

    location /.well-known/acme-challenge {
            root /var/www/letsencrypt;
        }

## Reload and check Nginx

    nginx -t && nginx -s reload
    
## Should output:

    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful

## Next ...

    mkdir /var/www/letsencrypt
    cd /opt/letsencrypt
    ./letsencrypt-auto --config /etc/letsencrypt/configs/domain.tld.conf certonly

## Comment out and add the following to nginx config below:
  
    nano  /etc/nginx/sites-available/fusionpbx

         #ssl_certificate         /etc/ssl/certs/nginx.crt;
         #ssl_certificate_key     /etc/ssl/private/nginx.key;
         ssl_certificate /etc/letsencrypt/live/domain.tld/fullchain.pem;
         ssl_certificate_key /etc/letsencrypt/live/domain.tld/privkey.pem;
       
## Now create script to renew cert:

    cd /etc/fusionpbx
    nano renew-letsencrypt.sh

## Example renew-letsencrypt.sh script

    #!/bin/sh

    cd /opt/letsencrypt/
    ./certbot-auto --config /etc/letsencrypt/configs/my-domain.conf certonly --non-interactive --keep-until-expiring --agree-tos --quiet

    if [ $? -ne 0 ]
     then
            ERRORLOG=`tail /var/log/letsencrypt/letsencrypt.log`
            echo -e "The Let's Encrypt cert has not been renewed! \n \n" \
                     $ERRORLOG
     else
            nginx -s reload
    fi


## Now make the script executable and create cronjob to run it once per week

    chmod +x renew-letsencrypt.sh

    crontab -e
    
## Add this add end of crontab file

    30 2 * * 1 /etc/fusionpbx/renew-letsencrypt.sh

# Disable or Remove IPv6 if not needed

    Web GUI: Advanced > SIP Profiles > interal-ipv6
    Web GUI: Advanced > SIP Profiles > external-ipv6

# Security Recommendation - Limit access to xml phone configuration directories by  nginx directory IP address whitelist

## Create whitelist called admin-ips:

    mkdir /var/www/fusionpbx/xml
    chown www-data:www-data /var/www/fusionpbx/xml

    nano /etc/nginx/includes/admin-ips

    allow 1.2.3.4;

    nano /etc/nginx/includes/customer1-ips

    allow 4.4.4.4;

    nano /etc/nginx/sites-enabled/fusionpbx

## In the server Listen 80 section, add this to the end:

    location ^~ /app/provision/ {
        include /etc/nginx/includes/admin-ips;
        deny all;
    }

    location ^~ /xml/ {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        deny all;
    }
    location ^~ /xml/customer1/ {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        include /etc/nginx/includes/customer1-ips;
        deny all;
    }
 
Here I am making a firmware directory called /fw, and I don't care who reads those files, it will just be used for phones to download firmware updates

    location ^~ /fw/ {
        root /var/www/fusionpbx;
        allow all;
    }
    service nginx reload
    
Now the directories are secured per customer using IP address whitelists. We also to make a modification to nginx to not force HTTPS on these directories because, unfortunately, some phones do not support HTTPS config download. Here is how to do that (this only needs to be applied under the Listen 80 server directive):
    
    listen 80;
    server_name fusionpbx;
    
    set $allow_http 0;
    
    if ($uri ~* ^.*provision.*$) {
        set $allow_http 1;
    }
    
    if ($uri ~* ^.*xml.*$) {
        set $allow_http 1;
    }
    
    if ($uri ~* ^.*fw.*$) {
        set $allow_http 1;
    }
    
    if ($allow_http = 0) {
        rewrite ^(.*) https://$host$1 permanent;
        break;
    }
    
## In the server Listen 443 section, add this to the end:
This allows only admins to access the xml directory, but customer1 can access /xm/customer1 where that customer's configs are. If I later set up customer2, they wouldn't be able to access configs in directories other than their own.

    location ^~ /app/provision/ {
        include /etc/nginx/includes/admin-ips;
        deny all;
    }

    location ^~ /xml/ {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        deny all;
    }
    location ^~ /xml/customer1/ {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        include /etc/nginx/includes/customer1-ips;
        deny all;
    }

Here I am making a firmware directory called /fw, and I don't care who reads those files, it will just be used for phones to download firmware updates

    location ^~ /fw/ {
        root /var/www/fusionpbx;
        allow all;
    }
    service nginx reload
    
# Configure DHCP option 66 at phone location
DHCP option 66 is used for zero-touch provisioning. A factory reset phone will grab DHCP and if option 66 is included it will use that to contact the provisioning server to download its configuration, upgrade firmware etc. DHCP option 66 is usually not available on consumer routers unless Linux firmware is used. Windows servers also have the option available.

## DHCP Option 66 configuration on ASUS Merlin

From web GUI:

    Administration > System > Enable JFFS partition > Yes
    Administration > System > Format JFFS partition at next boot > No
    Administration > System > Enable SSH > Yes
    
From SSH

    cd /jffs/configs
    vi dnsmasq.conf.add
        dhcp-option=66,"http://yourpbx.yourdomain.com"
        
Save that then reload dnsmasq:

    service restart_dnsmasq

# Method to enable global username 
## Allows you to log into any domain from any URL as long as username is unique

    Advanced > default settings -> Add > category: user
    subcategory: unique
    type: text
    value: global
    hit the reload button
    now usernames have to be unique across the entire system for it to work
    
