# Raspberry Pi Node Server - The complete instructions

So you just bought a new Raspberry Pi and you want to set it up as a web server? And you want to use Node.js? And have access to MongoDB?

Congrats, you're in the right place. You're only 10 steps away from Raspberry Pi-Node-Mongo goodness, and three of those steps are optional. Let's get started.

## Steps
1. [Install Raspbian onto SD Card](#step-1)
2. [Boot from USB (optional)](#step-2)
3. [Initial Configuration](#step-3)
4. [Internal Networking](#step-4)
5. [Install all the things](#step-5)
6. [Get your code, Run your code](#step-6)
7. [Proxy requests with nginx](#step-7)
8. [Make your web server available from ... the web.](#step-8)
9. [Enable SSL like a boss (optional)](#step-9)
10. [Enable Cross Origin Requests (optional)](#step-10)
11. [Step 11: BONUS! Control GPIO pins with Node.js](#step-11)

## <a name="step-1"></a> Step 1: Install Raspbian onto SD Card
On your development machine, [Download Raspbian Lite](https://www.raspberrypi.org/downloads/raspbian/). Unzip that file to get the `.img`.

If your dev machine is Windows or Linux, you can lookup [directions online](https://www.raspberrypi.org/documentation/installation/installing-images/README.md).

If you are using a Mac:
Format SD card, MS-DOS (FAT) using Disk Utility. Run `diskutil list` from terminal to get number of SD Card disk.

Then to copy the OS to the SD card, run (changing the location and filename of the .img file you just downloaded):
```
sudo dd bs=1m if=~/Downloads/2017-03-02-raspbian-jessie-lite.img of=/dev/rdisk{NUMBER_OF_SD_CARD_DISK}
```
After a while, you should see output similar to

```
1329+0 records in
1329+0 records out
1393557504 bytes transferred in 367.987715 secs (3786967 bytes/sec)
```

Once done, eject the SD card by running.
```
sudo diskutil eject /dev/disk{NUMBER_OF_SD_CARD_DISK}
```

Now plug that SD card, a montior and a keyboard into your Raspberry Pi, power it up and dive in.

*Note* - By default Raspbian ships with the British keyboard layout, which makes some keys hard to find. If you can't find the `|` character, ensure Num Lock is on, hold alt, then press 1,2,4 on the keypad.

## <a name="step-2"></a>Step 2:  Boot from USB (optional)
If your paranoid about the write limits of SD cards over time, or have been burned by a corupted SD card one too many times, you can instruct your Pi to boot from a USB flash drive instead. If you're not worried about this, skip ahead to [Step 3](#step-3).

If you have a Raspberry Pi 3, there are [new instructions](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/msd.md) on how to accomplish this. The cool thing about this, is that after following those steps, you can remove the SD and boot entirely from USB. Pretty cool.

Login using username `pi` and password `rapsberry`, then:
```
sudo apt-get update && sudo apt-get upgrade
echo program_usb_boot_mode=1 | sudo tee -a /boot/config.txt
sudo reboot
```
Log back in as `pi`, and run the following command:

```
vcgencmd otp_dump | grep 17:
```

If that outputs `17:3020000a`, congrats, you win.

Now you need to add Raspbian to your USB drive. On your dev machine, repeat [Step 1](#step-1), only with the USB drive, not the SD card. Once that finishes up, shutdown the Raspberry Pi (`sudo shutdown -h now`), eject the SD card, insert your USB drive and plug her in.

If all went as planned your Pi should startup as normal after 5-10 seconds and you're now done with the SD card for good!

If you have a Pi older than 3, you can still run your server off of a USB stick, but you'll still need the SD card for booting up. Instructions for that can be [found here](http://jonathanmh.com/boot-raspberry-pi-from-a-usb-stick/).

## <a name="step-3"></a> Step 3) Initial Configuration
Using `sudo raspi-config`, change hostname, enable SSH, wait for network on boot, reduce GPU to 16MB.

Optionally, you can edit the message that appears in the terminal when you first log in. I like to add some [ascii art](http://patorjk.com/software/taag/) to match the Pi's host name. To do so, run:

```
sudo nano /etc/motd
```

and paste in the message you want displayed.

### Setup new user
It's generally a good idea to have a different linux user for each app running on your server. To create a new user, for an `api` app for example:
```
sudo useradd -m api -G sudo
sudo passwd api
logout
```
Login with the user and password you just created. After you've confirmed your new user has sudo privelages, you can delete the default `pi` user for good measure:
```
sudo deluser pi
```

## <a name="step-4"></a> Step 4) Internal Networking
You can plug your Pi via ethernet cable to your router or setup Wifi.

To connect to Wifi: `sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`. Add the following:
```
network={
ssid="SSID"
psk="WIFI PASSWORD"
}
```

Most wireless routers these days are also DHCP servers, which means by default, you will get a different IP address every time you connect to the network. This isn't ideal, as you'll need to know where to find your server on the network if you're going to route requests to or SSH into it.

To assign Static IP address to your Pi on the internal network:

```
sudo nano /etc/dhcpcd.conf
```
 Add the following:
```
interface wlan0

# Desired IP Address. Keep /24
static ip_address=192.168.0.200/24
# IP Address of router
static routers=192.168.0.1
# IP Address of router
static domain_name_servers=192.168.0.1
```

Restart Wifi:
```
sudo ifdown wlan0
sudo ifup wlan0
```

If that is done correctly, you can now connect to your Pi via SSH on the local network. What that means is you can ditch the keyboard and monitor that are plugged into your Pi, and do the rest of the work from your normal dev machine. Woo!

From the terminal on your dev machine (or whatever SSH client you use on Windows), you can run

```
ssh api@192.168.0.200
```
Use the password you set when you created the new user.

## <a name="step-5"></a> Step 5) Install all the things:

Pull in reference to Node v6 (latest LTS version). Change version digit in the URL if you want a different version (https://deb.nodesource.com/setup_7.x for example)
```
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
```

Install the things
```
sudo apt-get update
sudo apt-get install git nodejs nginx -y
```
Install will take a decent amount of time to run (10-20 minutes)

Install yarn and pm2 with npm
```
sudo npm install -g yarn pm2
```

### Configure PM2 Logrotate (optional)
PM2, the process manager that keeps your Node scripts running, outputs great log files. However, without configuration these log files can eat up more and more storage on you Pi. The `pm2-logrotate` module can help with this issue.

```
# Install the module
pm2 install pm2-logrotate
# Keep at most 90 days of logs
pm2 set pm2-logrotate:retain 90
# Gzip old log files to save space
pm2 set pm2-logrotate:compress true
```

### Upgrade Nginx (optional)
Installing Nginx in the normal way like we just did will install a much older version of the software, something like 1.6. If you are going want to do modern things, including taking advantage of the HTTP/2 protocol, you're going to need a newer version of Nginx (at least 1.9.5). If you're fine with the older version, feel free to skip this section.

After installing nginx in the normal way above, remove it with `apt-get`:

```
sudo apt-get remove nginx
```

We are going to download and compile the newest version of Nginx, along with OpenSSL and PCRE. Luckily, a guy by the name of [Matt Wilcox]() has written [a script to do exactly that on a Raspi](). Download that script via wget:

```
cd ~
wget https://gist.githubusercontent.com/MattWilcox/402e2e8aa2e1c132ee24/raw/1a4f1f78cb69b715d7e63a6c9b97032bc9ddebd8/build_nginx.sh
```

Check for the latest version numbers of [OpenSSL](https://www.openssl.org/source/) (version 1.0.2d), [PCRE](http://www.pcre.org/) (version 8.39) and [Nginx](http://nginx.org/en/download.html) (version 1.9.7). The latest versions I could get to work are in parenthesis.

Edit variables at the top of the build script we just downloaded to reflect these version numbers.

```
sudo nano ~/build_nginx.sh
```

Make the file executable and run it:

```
sudo chmod +x build_nginx.sh
sudo ./build_nginx.sh
```

This will take a while to run. Be chill.

### Tweak Nginx (optional)
By default Nginx will advertise it's exact version number, in the `server` header and a couple other places. You can stop this behavior by editing the nginx config file:

```
sudo nano /etc/nginx/nginx.conf
```
and ensuring the following line is NOT commented out:
```
server_tokens off;
```

Nginx also ships with a default site that just shows a "Welcome to nginx on Debian!" page. This isn't really useful to us, and since we'll be adding new nginx sites, we can disable this default site. To do so, run:

```
sudo nano /etc/nginx/sites-available/default
```
and add `return 404;` in the line right after the listen directives.

Reload nginx to see our changes:
```
# Test your changes
sudo nginx -t
# If that says OK, reload
sudo service nginx restart
```

### MongoDB
If you want mongodb v2.4, just run:

```
sudo apt-get install mongodb
```

If you want v3.0.14 (which is the highest version of Mongo that supports a 32bit system), [do the following](http://andyfelong.com/2017/03/mongodb-3-0-14-binaries-for-raspberry-pi-3/):

Check for mongodb user
```
grep mongodb /etc/passwd
# if NO mongodb user, create:
sudo adduser --ingroup nogroup --shell /etc/false --disabled-password --gecos "" \
--no-create-home mongodb
```

Download Binaries
```
sudo apt-get upgrade
mkdir ~/mongo
# Core Mongo
wget -P ~/mongo/ http://andyfelong.com/downloads/core_mongodb_3_0_14.tar.gz
# Mongo Tools
wget -P ~/mongo/ http://andyfelong.com/downloads/tools_mongodb_3_0_14.tar.gz

# Unzip Core files:
tar -xvzf core_mongodb_3_0_14.tar.gz

# Move the files where they need to go
sudo chown root:root mongo*
sudo chmod 755 mongo*
sudo strip mongo*
sudo cp -p mongo* /usr/bin

# Log and config directories
sudo mkdir /var/log/mongodb
sudo chown mongodb:nogroup /var/log/mongodb
sudo mkdir /var/lib/mongodb
sudo chown mongodb:root /var/lib/mongodb
sudo chmod 775 /var/lib/mongodb

cd /etc
sudo nano mongodb.conf
```
Add the following text to mongdb.conf:

```
bind_ip = 127.0.0.1
quiet = true
dbpath = /var/lib/mongodb
logpath = /var/log/mongodb/mongod.log
logappend = true
storageEngine = mmapv1
```
__The wiredTiger storage engine isn't supported on the 32-bit ARM chip. :'(__

Add to `service`:

```
cd /lib/systemd/system
sudo nano mongodb.service
```

Add the following text to `mongodb.service`:
```
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongodb
ExecStart=/usr/bin/mongod --quiet --config /etc/mongodb.conf

[Install]
WantedBy=multi-user.target
```

Add the following line to `~/.bashrc` (it's a setting that mongo needs to run):

```
export LC_ALL=C
```

Start up mongo!
```
sudo systemctl unmask mongodb
sudo service mongodb start
```

Enable all the mongo tools:

```
cd ~/mongo
mkdir tools
tar -xzvf tools_mongodb_3_0_14.tar.gz --directory tools/
cd tools
sudo strip ./*
sudo chown root:root ./*
sudo chmod 755 ./*
sudo mv ./* /usr/bin
```
Mongo 3 on your Raspi 3. Boomshakalaka.

## <a name="step-6"></a> Step 6) Get your code, Run your code
Generate SSH Key
```
ssh-keygen -f ~/.ssh/id_rsa -C "id_rsa"
```
In a browser, go to Github or Bitbucket and Add SSH Key OR Deploy/Access Key in your repository. Paste in output from:
```
cat ~/.ssh/id_rsa.pub
```

Clone your repository into your home dir.

```
cd ~ && git clone git@URL_OF_REPOSITORY.git
```

If needed, add app config env vars to .bashrc
```
sudo nano ~/.bashrc
# export APP_CONFIG_OPTION=foobar
```

Run your code with PM2:
```
pm2 start ~/APP_DIRECTORY/index.js
```
Make sure your app starts up when raspi restarts:

```
pm2 startup
## This command will tell you to run another command as sudo. Do that.
pm2 save
```

If you run into issues with your PM2 not starting on reboot, even after running the above commands, you can add the following cronjob: `crontab -e`

```
@reboot cd ~/APP_DIRECTORY/ && pm2 start index.js
```

Your app should now be available to any computer on your local network. You can visit

[192.168.0.200:3000](http://192.168.0.200:3000)

and you'll see your app running (where the IP is the static internal IP address you assinged the PI, and the port is port your Node.js code is litening on).

## <a name="step-7"></a> Step 7) Proxy requests with nginx
You're node app most likely isn't running on port 80. It turns out it's kind of a pain to get node servers to directly listen on port 80. Enter nginx.

Add new nginx site block:
```
sudo nano /etc/nginx/sites-available/{APP_NAME}
sudo service nginx reload
```

Add the following text:

```
upstream {APP_NAME} {
    server 127.0.0.1:{NODE_APP_PORT};
    keepalive 64;
}

server {
    listen 80;

    server_name {DESIRED_DOMAIN_NAME};

    location / {
        proxy_pass http://{APP_NAME}/;
        proxy_http_version 1.1;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
        proxy_redirect off;
        proxy_read_timeout 240s;
    }
}
```

Enable newly configured site:

```
sudo ln -s /etc/nginx/sites-available/{APP_NAME} /etc/nginx/sites-enabled/
sudo service nginx reload
```

You can edit the hosts file (/etc/hosts) on your dev machine to point your newly added domain name to your Raspberry Pi's static internal IP to test your nginx config.

## <a name="step-8"></a> Step 8) Make your web server available from ... the web.

If you don't have a static IP from your internet provider (not sure? Odds are you don't), you'll need to use a Dynamic DNS service.

Head over to [Duck DNS](http://www.duckdns.org/). It's free and awesome. Make an account and claim a domain. Follow instructions on [this page](https://www.duckdns.org/install.jsp?tab=linux-cron).

```
mkdir ~/duckdns
cd ~/duckdns
nano duck.sh
```

Add the following line to `duck.sh`

```
echo url="https://www.duckdns.org/update?domains={YOUR_DOMAIN}&token={YOUR_TOKEN}4&ip=" | curl -k -o ~/duckdns/duck.log -K -
```
Make the file executable:
```
chmod 700 duck.sh
```

Run that file every 5 minutes. Edit your crontab (`crontab -e`) and add the following line:
```
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

Test the script:
```
./duck.sh
cat duck.log
```

If it's says `OK`, we good.

### Forward incoming requests to your network
 You'll need to hop into your router's admin screen and forward incoming requests on specific ports to your raspi. [PortForward.com/](https://portforward.com/) has more info on specifics on how to do this on various routers.

Forward only ports 80 (HTTP), 443 (HTTPS), and 22 (SSH) to the static internal IP you assigned your Pi.

Once that's done correctly, you should be able to go to [{APP_NAME}.duckdns.org](http://{APP_NAME}.duckdns.org) and see your app. Woo.


#### Use your own domain
Own a domain name? If you've spent this much time hacking on a raspberry pi, I'm going to bet you do.

You can add a CNAME record to access your site via a submain of the domain name you own.
The CNAME record should point to your {APP_NAME}.duckdns.org domain and the host should be your desired subdomain.

Once that change is saved and propogates, you should be able to access your Node.js app via a domain like:

[node-is-cool.myawesomedomain.com](http://node-is-cool.myawesomedomain.com)

### <a name="step-9"></a> Step 9) Enable SSL like a boss (optional)
Let's Encrypt makes enabling https free and relatively easy. You're already this deep in. Why stop now? [Link](https://www.linuxbabe.com/linux-server/install-lets-encrypt-free-tlsssl-certificate-nginx-debian-8-server)

```
sudo nano /etc/apt/sources.list
```
and add the following line to the end:

```
deb http://ftp.debian.org/debian jessie-backports main
```

Install the client:
```
sudo apt-get update
sudo apt-get install letsencrypt -t jessie-backports
```

Get a cert:

```
sudo service nginx stop
sudo letsencrypt certonly --email <your-email-address> -d <your-domain-name>
```

Select option 2, standalone.

Renew the cert automatically by running:

```
mkdir ~/log
sudo crontab -e
```

and pasting:

```
# Check for cert renewal every Monday at 2:30am
30 2 * * 1 /usr/bin/certbot renew >> /home/{username}/log/le-renew.log
# Reload Nginx every Monday at 2:35am, to use potential new cert
35 2 * * 1 /etc/init.d/nginx reload
```

Now we need to configure nginx to serve your app over SSL.

Generate strong Diffie–Hellman key (this will take a while to run):

```
sudo mkdir /etc/nginx/ssl/
sudo openssl dhparam -out /etc/ssl/certs/dhparam2048.pem 2048
```

Add SSL params config file:

```
sudo nano /etc/nginx/snippets/ssl-params.conf
```

And paste in:

```
ssl_protocols TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_dhparam /etc/ssl/certs/dhparam2048.pem;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains;" always;
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options nosniff;
add_header Referrer-Policy "no-referrer";
add_header X-XSS-Protection "1; mode=block";
```

Note: If you did not upgrade Nginx, remove the word `always` from the `Strict-Transport-Security` line, so it reads:

```
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains;";
```

See [cipherli.st](https://cipherli.st/) for explanations for all of these SSL settings.

Edit your nginx site config file:

```
sudo nano /etc/nginx/site-available/{APP_NAME}
```


Delete everything and paste in the following:
```
upstream {APP_NAME} {
    server 127.0.0.1:{NODE_APP_PORT};
    keepalive 64;
}

server {
    listen 80;
    server_name {DOMAIN_NAME};
    return 301 https://$host$request_uri;
}

server {

    listen 443 ssl http2;

    server_name {DOMAIN_NAME};

    ssl_certificate /etc/letsencrypt/live/{DOMAIN_NAME}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{DOMAIN_NAME}/privkey.pem;

    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://{APP_NAME}/;
        proxy_http_version 1.1;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
        proxy_redirect off;
        proxy_read_timeout 240s;
    }
}
```

Note: If you did not upgrade Nginx, remove `http2` from the `listen 443` line so it reads:
```
listen 443 ssl;
```

Test your nginx config:

```
sudo nginx -t
```

If all good, reload nginx:

```
sudo service nginx reload
```

## <a name="step-10"></a> Step 10: Enable Cross Origin Requests (optional)

If you're app is an API that will be called via ajax on other domains, we'll need to adjust our nginx site config to tell browsers that we know what we're doing.

```
sudo nano /etc/nginx/site-available/{APP_NAME}
```
Delete everything and paste in the following:

```
upstream {APP_NAME} {
    server 127.0.0.1:3111;
    keepalive 64;
}

server {
    listen 80;
    server_name {DOMAIN_NAME};
    return 301 https://$host$request_uri;
}

server {

    listen 443 ssl http2;

    server_name {DOMAIN_NAME};

    ssl_certificate /etc/letsencrypt/live/{DOMAIN_NAME}/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/{DOMAIN_NAME}/privkey.pem;

    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://{APP_NAME}/;
        proxy_http_version 1.1;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
        proxy_max_temp_file_size 0;
        proxy_redirect off;
        proxy_read_timeout 240s;

        if ($http_origin ~* (^https?://.*\.example\.com$|^https?://localhost:\d+$)) {
            set $cors "true";
        }

        # See https://gist.github.com/algal/5480916
        # for CORS setup
        # OPTIONS indicates a CORS pre-flight request
        if ($request_method = 'OPTIONS') {
            set $cors "${cors}options";
        }

        # non-OPTIONS indicates a normal CORS request
        if ($request_method = 'GET') {
            set $cors "${cors}normal";
        }
        if ($request_method = 'PUT') {
            set $cors "${cors}normal";
        }
        if ($request_method = 'PATCH') {
            set $cors "${cors}normal";
        }
        if ($request_method = 'DELETE') {
            set $cors "${cors}normal";
        }
        if ($request_method = 'POST') {
            set $cors "${cors}normal";
        }

        # if it's a normal request, set the standard CORS responses header
        if ($cors = "truenormal") {
            add_header 'Access-Control-Allow-Origin' "$http_origin";
            add_header 'Access-Control-Allow-Credentials' 'true';
            add_header 'Access-Control-Expose-Headers' 'accesstoken, refreshtoken, server-timing';
        }

        if ($cors = "trueoptions") {
            add_header 'Access-Control-Allow-Origin' "$http_origin";
            add_header 'Access-Control-Allow-Credentials' 'true';

            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Access-Control-Allow-Methods' 'GET, PUT, PATCH, DELETE, POST, OPTIONS';
            add_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,Keep-Alive,X-Requested-With,If-Modified-Since';

            add_header 'Content-Length' 0;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            return 204;
        }
    }
}
```

Change the following line:
```
if ($http_origin ~* (^https?://.*\.example\.com$|^https?://localhost:\d+$)) {
```

to use a regex for whatever domains you want to whitelist. In this example, we're allowing requests from any subdomain on `example.com` and requests from `localhost` on any port number, both on either `http` or `https`.

Checkout this [annotated config file](https://gist.github.com/algal/5480916) to learn more about each of the additions we're making.

## <a name="step-11"></a> Step 11: BONUS! Control GPIO pins with Node.js
One of the best things about using a Raspberry Pi is the ability to interact with the physical world using the [General Purpose Input/Output pins](https://www.raspberrypi.org/documentation/usage/gpio-plus-and-raspi2/). The RasPi 3 has 40 pins, 28 of which are programmable. You can wire these into buttons, switchs, relays, servos, sensors, etc to do any sort of cool thing you can dream up. [pinout.xyz/](https://pinout.xyz/) is a great resource to find out which pins do what.

There exist lots of Node.js libraries to help interact with these pins, but few are kept up to date and work with the latest RasPi models and newest versions of Node. However I found the [RPIO](https://www.npmjs.com/package/rpio) package library and it is fantastic. It has an easy-to-understand interface, is super performant, and supports pretty much all RasPi models and most Node.js versions, including 6.x and 7.x.

To use it in your Node.js app run:

```
yarn add rpio
```

Before you can you use the library in your Node.js app, you'll need to make sure the user running the Node script has the correct permissions.

Add the your user to the `gpio` group:

```
sudo usermod -a -G gpio USERNAME
```

And add a file to tell the interface to trust the `gpio` group:

```
sudo nano /etc/udev/rules.d/20-gpiomem.rules
```

and paste in:

```
SUBSYSTEM=="bcm2835-gpiomem", KERNEL=="gpiomem", GROUP="gpio", MODE="0660"
```

After restarting the Pi, you are ready to start controlling the pins with your app.

Include the library in your javascript file:

```
const rpio = require('rpio');
```

Setting a pin as output and toggling it to high is as easy:

```
rpio.open(12, rpio.OUTPUT, rpio.LOW);
rpio.write(12, rpio.HIGH);
```

And reading the state of an input pin is just as easy:

```
rpio.open(15, rpio.INPUT);
const state = rpio.read(15);
```

## You did it!

Congrats!

If you followed all the steps, you now have a Node.js app mangaged with PM2 with access to locally running MongoDB, being served by an SSL, HTTP/2 Nginx proxy server, that's available to the entire Internet while being hosted on a Raspberry Pi at your house.

What else could you want in life besides exactly that?
