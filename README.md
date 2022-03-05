# New York Times to PNG: nyt2png
## Display an auto-refreshing version of the New York Times front page on a 32" eInk display. 

Original article: https://alexanderklopping.medium.com/an-updated-daily-front-page-of-the-new-york-times-as-artwork-on-your-wall-3b28c3261478

Which was based on: https://onezero.medium.com/the-morning-paper-revisited-35b407822494

Device: https://www.visionect.com/product/place-and-play-32/

Visionconnect documentation: https://docs.visionect.com/VisionectSoftwareSuite/Installation.html#embedded-boards

## Web page rendering software

First, you have to install Visionect's server to translate a web page into a format compatible with the eInk display.
The software is supplied by Visionect and can be installed with the following instructions. 

Do a fresh install of Debian 11 with nothing but SSH enabled in the installer. 
Then perform the following commands as root...
```
apt-get update
apt-get -y install ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get -y install docker-ce docker-ce-cli containerd.io wget docker-compose
```
Copy the docker compose part of
https://hub.docker.com/r/visionect/visionect-server-v3-armhf/ for Raspberry Pi or
https://hub.docker.com/r/visionect/visionect-server-v3/ for x64 
to docker-compose.yml.  In the same directory as your docker-compose.yml file, and type
```
docker-compose up -d
```
On your laptop, open your browser to http://THE_SERVER_ADDRESS:8081
See notes below if you are having problems connecting to the visionect server.

---
Now you need to configure the display itself. 
Connect the mini serial cable between the back of the display and your laptop.
Under OSX you can check what the USB serial device is by
```
ls -la /dev/tty.*
```
and look for something in the form of tty.usbserial-AU02K26M
Using minicom, configure the serial port and baud rate speed, which is 115200
```
minicom -s 
```
Hit enter a few times and you should see a > prompt. 
To stop the boot looping and configure the basics
```
cs 0
wifi_ssid_set SSID
wifi_psk_set PASS
server_tcp_set YOUR_SERVER_ADDRESS 11113
flash_save
reboot
```
The screen should connect to the visionect server and you will see it register itself in the dashboard.

#########################
# then we need to get the web server up and running for the NY Times page
# the script is based on this https://gist.github.com/alexandernl/8090a79a7af4197c5f5571e8cc8c05b8
# first install the tools
git clone <this repo>
apt -y install nginx imagick php-fpm
mkdir /var/www/html/
chmod 766 /var/www/html/nyt2png

# Edit /etc/nginx/sites-available/default to the following
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;
	index index.php index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		try_files $uri $uri/ =404;
	}

	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/run/php/php7.4-fpm.sock; # PHP version (php -v command)
	}
}

# Then check the config
nginx -t
# Seeing no problems, reload nginx
service nginx reload

# Edit the /etc/ImageMagick-6/policy.xml to change...
<policy domain="coder" rights="none" pattern="PDF" />
# to...
<policy domain="coder" rights="read|write" pattern="PDF" />
# and...
<policy domain="module" rights="none" pattern="{PS,PDF,XPS}" />
# to...
<policy domain="module" rights="read|write" pattern="{PS,PDF,XPS}" />

# Reboot the machine since I don't know how to make the policy.xml changes take effect. 
reboot

#########################
# Upon reboot, you should be able to hit THE_SERVER_ADDRESS in your laptop browser and after fifeteen to thirty seconds you should see a stretched out version of the New York Times web page in your browser.
# If it doesn't show up, you can do a...
tail -f /var/log/nginx/* 
# and watch for errors and access.


# NOTES
# To shell into the docker container you first need to figure out the name.
docker ps
# The container names will be to the far left.  Then you can open a shell to the container with... 
docker exec -it abc123 /bin/bash
# The logs will be in /var/log/vss
tail -f /var/log/vss/*
