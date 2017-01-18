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
    go to advanced -> default settings >  menu_style >  set to inline

# Lets Encrypt Install

    cd /usr/src/
    git clone https://github.com/letsencrypt/letsencrypt /opt/letsencrypt
    cd /opt/letsencrypt
    chmod a+x ./certbot-auto
    ./certbot-auto
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
    location ^~ /xml/customer1 {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        include /etc/nginx/includes/customer1-ips;
        deny all;
    }
    
## In the server Listen 443 section, add this to the end:

    location ^~ /app/provision/ {
        include /etc/nginx/includes/admin-ips;
        deny all;
    }

    location ^~ /xml/ {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        deny all;
    }
    location ^~ /xml/customer1 {
        root /var/www/fusionpbx;
        include /etc/nginx/includes/admin-ips;
        include /etc/nginx/includes/customer1-ips;
        deny all;
    }

    service nginx reload

# Method to enable global username 
## Allows you to log into any domain from any URL as long as username is unique

    Advanced > default settings -> Add > category: user
    subcategory: unique
    type: text
    value: global
    hit the reload button
    now usernames have to be unique across the entire system for it to work

