# New York Times to PNG: nyt2png
## Display an auto-refreshing New York Times front page on a 32" eInk display. 

![IMG_0988](https://user-images.githubusercontent.com/515629/156884064-3f54a3ef-858b-414c-8180-81b57675f18c.jpg)


This project was born from a fantastic lust for the New York Times projects by [Max Braun](https://onezero.medium.com/the-morning-paper-revisited-35b407822494) and later by [Alexander Klöpping](https://alexanderklopping.medium.com/an-updated-daily-front-page-of-the-new-york-times-as-artwork-on-your-wall-3b28c3261478). Their motivations for a minimalist but informative display resonated with me completely. Both articles were striking existence proofs but left a lot to be desired regarding technical detail on software. Max was held under NDA for parts of his software implementation, and Alexander states that he had a friend do the script. As a result, there was little information about tying all of these software bits together.   

I agreed with Alexander's sentiment on not wanting to spend a lot of time on the hardware and opted to pick up one of the commercially available [Visionect 32" displays](https://www.visionect.com/product/place-and-play-32/). This display is beautifully constructed and has no wires. It is hard to overstate the crispness of the images rendered on the screen. Aside from the glass in front of the eInk display, the letters are indistinguishable from printed text. It takes a few seconds for a page to render, but the results are positively sublime once it is done.  

Everything worked by the end of the project, but it was quite a journey. So this is the abridged solution after hours of frustration. I currently have this running in a virtual machine on a server on my home network. I previously had it running on a Raspberry Pi 4 until the SD card died, and I had to remember how to install everything. 

These instructions should work equally well on a Raspberry Pi or x64 hardware. However, be patient if you decide to run it on a Raspberry Pi. Rendering the PDF to a PNG can sometimes take around 60 seconds, and it will consume all of the CPU and memory on the Pi while creating the PNG in memory and converting it in Visionect's software for the eInk display. So while it is feasible, I don't plan on running it on a Pi again. 

I am running the Visionect software and the Nginx web server on the same virtual machine for the instructions below. It is a minimal Debian 11 install, but I am sure that Ubuntu would also work. The raw VM disk is 10GB and uses the default layout with 1GB of swap. When the system is installed, booted, and idling, you will find 50% of the disk free and 50% of the swap available. After installation, you can trim the virtual machine down to one CPU core and 800MB of memory. That virtual machine configuration will leave you with enough headroom for updates and other activities in the background while a new web page rendering occurs.  

***

Original project by Max Braun: https://onezero.medium.com/the-morning-paper-revisited-35b407822494  
Updated project by Alexander Klöpping: https://alexanderklopping.medium.com/an-updated-daily-front-page-of-the-new-york-times-as-artwork-on-your-wall-3b28c3261478  
Device: https://www.visionect.com/product/place-and-play-32/  
Visionect documentation: https://docs.visionect.com/VisionectSoftwareSuite/Installation.html     
Original PDF to PNG php page: https://gist.github.com/alexandernl/8090a79a7af4197c5f5571e8cc8c05b8

***

## Web page rendering software

You have to install Visionect's server to translate a web page into a format compatible with the eInk display.  The software is supplied by Visionect as a docker container.  Do a fresh install of Debian 11 with nothing but SSH enabled in the installer.  

Lets install docker first.  Perform the following commands as root.
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
to docker-compose.yml.  In the same directory as your docker-compose.yml file type
```
docker-compose up -d
```
On your laptop, open your browser to http://YOUR_SERVER_ADDRESS:8081

See notes below if you are having problems connecting to the visionect server.

***

## Configure the display
Now you need to configure the display itself.  Connect the supplied USB cable between the back of the display and your laptop.
Under OSX you can check what the USB serial device is by
```
ls -la /dev/tty.*
```
and look for something in the form of tty.usbserial-AU02K26M.  
Using minicom, configure the serial port and baud rate speed, which is 115200
```
minicom -s 
```
After configuring the settings and entering minicom's terminal, hit enter a few times and you should see a > prompt. 
To stop the boot looping and configure the basics do the following.
```
cs 0
wifi_ssid_set SSID
wifi_psk_set PASS
server_tcp_set YOUR_SERVER_ADDRESS 11113
flash_save
reboot
```
The screen should connect to the visionect server and you will see it register itself in the dashboard.

***

## Configure the Web server for the New York Times page
We need to get the web server up and running for the NY Times page.  

First install the tools
```
apt -y install nginx imagick php-fpm git
cd ~
git clone https://github.com/mmicire/nyt2png.git
mkdir /var/www/html/
chmod 766 /var/www/html/nyt2png
cp ./nyt2png/index.php /var/www/html/
```

Edit /etc/nginx/sites-available/default to the following
```
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
```
Then check the config
```
nginx -t
```
If you see no errors, reload nginx
```
service nginx reload
```
Edit the /etc/ImageMagick-6/policy.xml to change
```
<policy domain="coder" rights="none" pattern="PDF" />
```
to
```
<policy domain="coder" rights="read|write" pattern="PDF" />
```
and
```
<policy domain="module" rights="none" pattern="{PS,PDF,XPS}" />
```
to
```
<policy domain="module" rights="read|write" pattern="{PS,PDF,XPS}" />
```

Reboot the machine (since I don't know how to make the policy.xml changes take effect otherwise). 
```
reboot
```

***

## Test the server
Upon reboot, you should be able to hit http://YOUR_SERVER_ADDRESS in your laptop browser and after fifeteen to thirty seconds you should see a stretched out version of the New York Times web page in your browser.  The image may look stretched because it is formatted to stretch to the proportions of the eInk display automatically.  Don't worry as long as something shows up.  If it doesn't show up, on the server you can run
```
tail -f /var/log/nginx/* 
```
and watch for errors and access.

***

## Final configuration

Finally, go into the Visionect configuration software at http://YOUR_SERVER_ADDRESS:8081 and in the display setting you should change the URL for the display to http://YOUR_SERVER_ADDRESS.  You do not need to set a refresh for the page in the control panel; the page has an auto-refresh configured itself and it has worked reliably for me. 

***

## ADDITIONAL NOTES
The Visionect servers can be very fragile and tempermental in my experience.  Once you get it running things are fine, but getting there can be frustrating.  Sometimes it is helpful to jump into the docker image and figure out what is breaking.  To shell into the docker container you first need to figure out the name.
```
docker ps
```
The container names will be to the far left.  Then you can open a shell to the container with
```
docker exec -it CONTAINER_NAME /bin/bash
```
The logs will be in /var/log/vss
```
tail -f /var/log/vss/*
```
