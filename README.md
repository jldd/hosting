General Information:
--------------------

This is a setup for a Tor based shared hosting server. It is provided as is and before putting it into production you should make changes according to your needs. This is a work in progress and you should carefully check the commit history for changes before updating.

Installation Instructions:
--------------------------

The configuration was tested with a standard Debian buster and Ubuntu 18.04 LTS installation. It's recommended you install Debian buster (or newer) on your server, but with a little tweaking you may also get this working on other distributions and/or versions. If you want to build it on a raspberry pi, please do not use the raspbian images as several things will break. Download an image for your pi model from [https://raspi.debian.net/daily-images/](https://raspi.debian.net/daily-images/) instead.

Uninstall packages that may interfere with this setup:
```
DEBIAN_FRONTEND=noninteractive apt-get purge -y apache2* resolvconf eatmydata exim4* imagemagick-6-common mysql-client* mysql-server* nginx* libnginx-mod* php7* && systemctl disable systemd-resolved.service && systemctl stop systemd-resolved.service
```

If you have problems resolving hostnames after this step, temporarily switch to a public nameserver like 1.1.1.1 (from CloudFlare) or 8.8.8.8 (from Google):

```
rm /etc/resolv.conf && echo "nameserver 1.1.1.1" > /etc/resolv.conf
```

Install custom optimized binaries
```
./install_binaries.sh
```

To get the latest mariadb version, you should follow these instructions to add the official repository for your distribution: (https://downloads.mariadb.org/mariadb/repositories/)

Add torproject to our repositories:
```
curl --socks5-hostname 127.0.0.1:9050 -sSL http://apow7mjfryruh65chtdydfmqfpj5btws7nbocgtaovhvezgccyjazpqd.onion/torproject.org/A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89.asc > /etc/apt/trusted.gpg.d/torproject.asc
echo "deb tor://apow7mjfryruh65chtdydfmqfpj5btws7nbocgtaovhvezgccyjazpqd.onion/torproject.org/ `lsb_release -cs` main" >> /etc/apt/sources.list
apt-get update && apt-get upgrade
```

Note that debian also has an onion service package archive, so you may want to edit /etc/apt/sources.list to load from there instead:
```
deb tor://2s4yqjx5ul6okpp3f2gaunr2syex5jgbfpfvhxxbbjwnrsvbk5v3qbid.onion/debian `lsb_release -cs` main
```

Copy (and modify according to your needs) the site files in `var/www` to `/var/www`, `usr/local` to `/usr/local`  and the configuration files in `etc` to `/etc` after installation has finished. Then restart some services:
```
systemctl daemon-reload && systemctl restart bind9.service && systemctl restart dnsmasq.service && systemctl restart tor@default.service
```

Now there should be an onion domain in `/var/lib/tor/hidden_service/hostname`:
```
cat /var/lib/tor/hidden_service/hostname
```

Replace the default domain with your domain in the following files:
```
/etc/postfix/sql/alias.cf
/etc/postfix/sender_login_maps
/etc/postfix/main.cf
/var/www/skel/www/index.hosting.html
/var/www/common.php
/etc/postfix/canonical
/etc/postfix-clearnet/canonical
```

In `/etc/postfix(-clearnet)/canonical` don't change the line that has `hosting.danwin1210.me` in it. It is a clearnet/tor address rewriting rule, and if you have your own clearnet domain, you should copy this and modify your copy to preserve sending mail to my host via tor and not via clearnet.

This setup has two postfix instances, one for receiving and sending mail to other .onion services and one for rewriting addresses to pass them on to a clearnet facing mail relay. You may or may not want to create the second instance by running
```
postmulti -e init
postmulti -I postfix-clearnet -e create
postmulti -i clearnet -e enable
postmulti -i clearnet -p start
```
If you created an instance, uncomment the clearnet relay related config in etc/postfix/main.cf and make sure to copy and modify the configuration files from etc/postfix-clearnet too

If you encountered the following issue: `postfix: fatal: chdir(/var/spool/postfix-clearnet): No such file or directory` you can just copy the chroot from the default postfix instance like this `cd /var/spool/ && cp -a postfix/ postfix-clearnet/`

After copying (and modifying) the posfix configuration, you need to create databases out of the mapping files (also each time you update those files):
```
postalias /etc/aliases
postmap /etc/postfix/canonical /etc/postfix/sender_login_maps /etc/postfix/transport
postmap /etc/postfix-clearnet/canonical /etc/postfix-clearnet/sasl_password /etc/postfix-clearnet/transport #only if you have a second instance
```

To save temporary files in memory, add the following to `/etc/fstab`:
```
tmpfs /tmp tmpfs defaults,noatime 0 0
tmpfs /var/log/nginx tmpfs rw,user,noatime 0 0
```

To harden the system and hide pids from non-root users, also add the following:
```
proc /proc proc defaults,hidepid=2 0 0
```

As time syncronisation is important, you should configure ntp servers in `/etc/systemd/timesyncd.conf` and make them match with the entries in `/etc/rc.local` iptables configuration

Enable the PHP-FPM default instances and nginx:
```
systemctl enable php7.4-fpm@default
systemctl enable php8.0-fpm@default
systemctl enable nginx
```

Edit `/etc/fstab` and add the `noatime,usrjquota=aquota.user,jqfmt=vfsv1` option to the `/home` mountpoint and `noatime`to `/`. Then initialize quota:
```
mount -o remount /home
quotacheck -cMu /home
quotaon /home
```

Install sodium_compat for v3 hidden_service support
```
cd /var/www && composer install
```

For web base database administration, check out the latest phpmyadmin and adminer:
```
cd /var/www/html/ && git clone -b STABLE https://github.com/phpmyadmin/phpmyadmin/ && cd phpmyadmin && composer install --no-dev && yarn
cd /var/www/html/ && git clone https://github.com/vrana/adminer/ && cd adminer && git submodule update --init
```

Once installed create a mysql user for phpmyadmin and cofigure it in `/var/www/html/phpmyadmin/config.inc.php` and fill `$cfg['blowfish_secret']` with random characters:
```
mysql
CREATE USER 'phpmyadmin'@'%' IDENTIFIED BY 'MY_PASSWORD';
CREATE DATABASE phpmyadmin;
GRANT ALL PRIVILEGES ON phpmyadmin.* TO 'phpmyadmin'@'%';
FLUSH PRIVILEGES;
quit
mysql phpmyadmin < /var/www/html/phpmyadmin/sql/create_tables.sql
```

For web based mail management grab the latest squirrelmail and install it in `/var/www/html/squirrelmail`:
```
cd /var/www/html/ && git clone https://github.com/RealityRipple/squirrelmail && cd squirrelmail && mkdir -p /var/www/data/squirrelmail/data /var/www/data/squirrelmail/attach && chown www-data:www-data -R /var/www/data && ./configure
```

Once it is downloaded, it will ask you for configuration. Things to change are:
```
D. > select dovecot
2. Server Settings > 1. Domain > Set your own .onion domain here
2. Server Settings > B. Update SMTP settings > 7. SMTP Authentication -> y -> plain -> n User are authenticated using their username + password
4. General Options > 1. Data Directory > /data/squirrelmail/data/
4. General Options > 2. Attachment Directory > /data/squirrelmail/attach/
4. General Options > 9. Allow editing of identity > n Users should not be able to fake email addresses > y They should be able to change display name > y They should be able to set a reply to mail > y additional headers are not required
10. Language settings > 4. Enable aggressive decoding
11. Tweaks > 2. Ask user info on first login > n (commonly confuses users)
11. Tweaks > 5. Use php iconv functions > y
```

Create a mysql user with all permissions for our hosting management:
```
mysql
CREATE USER 'hosting'@'%' IDENTIFIED BY 'MY_PASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'hosting'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
quit
```

Then edit the database configuration in `/var/www/common.php` and `/etc/postfix/sql/alias.cf`

Last but not least setup the database by running
```
php /var/www/setup.php
``` 

Enable systemd timers to regularly run various managing tasks:
```
systemctl enable hosting-del.timer && systemctl enable hosting.timer
```

Final step is to reboot wait about 5 minutes for all services to start and check if everything is working by creating a test account.
