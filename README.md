# how-to-secure-raspberrypi
A guide for secure your RaspberryPi from bad guys! It is usefult especially for those who want to expose RPi on the Internet (who don't?).
I assume that you are using the Raspbian distro.
All the operations will be done on terminal. Unless otherwise specified, the commands are executed on the Raspberry Pi terminal.
The guide has been tested with [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/) on [Raspberry Pi Model B+](https://www.raspberrypi.org/products/raspberry-pi-2-model-b/).
Any suggestion is welcome!
## Step 1: Manage users and passwords
### Change default password
The default user/password is:

- username: `pi`
- password: `raspberrypi`
        
Change `pi`'s password with:

    sudo passwd pi
        
And type the new password. Please, [choose a strong password... and remember it!!!](https://www.howtogeek.com/195430/how-to-create-a-strong-password-and-remember-it/)

### Change default user
This step is not mandatory, but usually it is a good idea to add a new user and disable `pi`, the default one.


- For create a new user, type:

        sudo adduser your-new-username

	and follow the instructions. You need to specify a password for the new user.

- Make it sudoer:

        sudo usermod -aG sudo your-new-username
        
- Switch to the new user using the password you defined before:
       
        su your-new-username

- Lock user `pi`:

        sudo usermod -L pi 

- Check that `pi` is locked:

        sudo passwd pi -S

	The second field of the output should be `L`.
	
### Check that root user is disabled

Usually, it should be disabled by default. 
In order to check that user `root` is disabled:

    sudo passwd root -S
    
It should print something like the following:
`root L 07/05/2017 0 99999 7 -1`

If the second field is equal `L`, it means that the user `root` is disabled (locked).
If not, then type the following command:
        
    sudo usermod -L root
    

    
It is not mandatory, but it is almost always a better idea to disable `root` and add an additional, unprivileged user to do common tasks, and when you need tu run a command with administrator privileges you can use `sudo`.

## Step 2: Configure the SSH server
Probably you arleady using a SSH-connection for run the above commands. If not, then I suggest you to use SSH in order to interact with the Rasperry Pi: it is very simple and helpful. Obviously, we need to configure the server correctly, especially if you want to access to your RPi from external networks.
### Install a SSH server
First of all, check if the SSH server is already installed and at the latest version.

    sudo apt-get install openssh-server 

Check if it is running

    sudo service ssh status
    
If not, then type and run the following command:

    sudo systemctl enable ssh

### Configure public key authentication
- First of all, **on your client terminal**, generate a new private-public key pair:

        ssh-keygen -b 4096

	Notice: if you already have a key pair (in `~/.ssh/`), the program should ask you if you want to override it. Usually it is not a good idea, because maybe you already used it in some way.
	If you want, you can set a passphrase, but it is not mandatory.
The newly-generated SSH keys are located in the `~/.ssh/` directory. You will find the private key in the `~/.ssh/id_rsa` file and the public key in the `~/.ssh/id_rsa.pub`file.

- Copy your public key to the server:

        ssh-copy-id <username>@<host>
	Now in the `~/.ssh/` folder there should be a new file called `authorized-keys`, with the public key of the client where you executed the above commands.

- Add private key identities to the authentication agent:

		ssh-add

- Now test the connection with:

        ssh <username>@<host>

For throubleshooting, I suggest to read the following article: 
https://help.ubuntu.com/community/SSH/OpenSSH/Keys

### Install fail2ban
If you use only public key authentication from the defined clients, then you can disable password authentication modifying the ssh configuration file:

    sudo nano /etc/ssh/sshd_config 

And replace the line `#PasswordAuthentication yes` with `PasswordAuthentication no`.

If, however, you want to use different clients, then you have to enable password authentication.
In this case, I **strongly reccomend** to use [fail2ban](http://www.fail2ban.org/wiki/index.php/Main_Page), which adds a firewall rule to block multiple log-in attempts, preventing dictionary attacks. After the number of attempts you specify, it will block for a time any more attempts from that IP address.
Further instructions and tips [here](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04).

- Install fail2ban:

        sudo apt-get install fail2ban

- Create `jail.local` configuration file, that you will use for override default configuration in `jail.conf`

		awk '{ printf "# "; print; }' /etc/fail2ban/jail.conf | sudo tee /etc/fail2ban/jail.local

Now you can override all default configurations by removing comments and modifying values.

## Step 3: Use a Dynamic DNS service
Now we will set up a DDNS in order to allow a SSH connection from... Everywhere (almost)!
In this guide, I will use [No-IP](https://www.noip.com/), and in the following I will assume that you already have an account and a configured domain name.
I mainly followed this [tutorial](http://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client/) and [this one](https://www.howtoforge.com/how-to-install-no-ip2-on-ubuntu-12.04-lts-in-order-to-host-servers-on-a-dynamic-ip-address).
Please, remember that before trying to access to the server from external networks, you need to configure your home router correctly (i.e. open external ports, define the port-forwarding rule etc.).

Execute this commands, following the prompted instructions:
    
    sudo cd /usr/local/src
    sudo wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
    sudo tar xzf noip-duc-linux.tar.gz
    sudo cd no-ip-2.1.9
    sudo make
    sudo make install

You'll probably end with this output:
	
	New configuration file '/tmp/no-ip2.conf' created.
	mv /tmp/no-ip2.conf /usr/local/etc/no-ip2.conf

Now, we need to make it run automatically at every startup.
Create the script that will run the service:

	sudo nano /etc/init.d/noip
Copy and paste the following script:

	#! /bin/sh
	case "$1" in
	    start)
		echo "Starting noip2"
		/usr/local/bin/noip2
	    ;;
	    stop)
		echo -n "Shutting down noip2"
		for i in `noip2 -S 2>&1 | grep Process | awk '{print $2}' | tr -d ','`
		do
		  noip2 -K $i
		done
	    ;;
	    *)
		echo "Usage: $0 {start|stop}"
		exit 1
	esac
	exit 0

You can test the script with:

	sudo /etc/init.d/noip stop
	sudo /etc/init.d/noip start


Change permissions:

	sudo chmod 700 /usr/local/bin/noip2
	sudo chown root:root /usr/local/bin/noip2
	sudo chmod 700 /etc/init.d/noip
	sudo chown root:root /etc/init.d/noip
	sudo chmod 700 /usr/local/etc/no-ip2.conf
	sudo chown root:root /usr/local/etc/no-ip2.conf


Now we will add the the noip start script to the ubuntu startup process

	sudo nano /etc/rc.local

In the line above "exit 0" add the following line: `/etc/init.d/noip start`

Reboot:

	sudo reboot

Test if the process is active:

	/usr/local/bin/noip2 -S

## Step 4: Install a Web Server (Nginx)
Run this command:

	sudo apt-get install nginx php5-fpm php5-curl php5-gd php5-cli php5-mcrypt php5-mysql php-apc mysql-server
	

- Nginx

	For enable/disable nginx at startup:
		
		sudo systemctl (disable|enable) nginx.service
		
	For start/stop/restart nginx service:
	
		sudo service nginx (start|stop|restart)

	Configuration files in `/etc/nginx/`.
	
	Now we will modify some configuration file.
	
		sudo nano /etc/nginx/nginx.conf
	
	1. Inside the `http { … }` block we want to un-comment the `server_tokens off` line;

		Enabling this line stops nginx from reporting which version it is to browsers; which helps to stop people from learning your nginx version and then Googling for exploits they might be able to use to hack you.
	
	2. Uncomment `#server_names_hash_bucket_size 64;`
	[Details](https://gist.github.com/muhammadghazali/6c2b8c80d5528e3118613746e0041263).
	
	3. We'll also harden nginx against DDOS attacks. Add these lines inside the http block, just before the end bracket }:
	
			##
			# Harden nginx against DDOS
			##

			client_header_timeout 10;
			client_body_timeout   10;
			keepalive_timeout     10 10;
			send_timeout          10;
	
- Enable PHP (check [here](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-12-04)):

	- Create `fastcgi_params`:

			sudo nano /etc/nginx/fastcgi_params
		
		and make sure it is the same of as described [here](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/) (in particular, check the HTTPS line).
	
	- Configure PHP:
	
		 	sudo nano /etc/php5/fpm/php.ini
 		
 	- Find the line, cgi.fix_pathinfo=1, and change the 1 to 0.

			cgi.fix_pathinfo=0

		If this number is kept as 1, the php interpreter will do its best to process the file that is as near to the requested file as possible. This is a possible security risk. If this number is set to 0, conversely, the interpreter will only process the exact file path—a much safer alternative. Save and Exit. 
	
	- Open the php5-fpm file configuration,`www.conf`:

			sudo nano /etc/php5/fpm/pool.d/www.conf

	- Find the line, `listen = 127.0.0.1:9000`, and change the `127.0.0.1:9000` to `/var/run/php5-fpm.sock`.

			listen = /var/run/php5-fpm.sock



	- Restart php-fpm:

		sudo service php5-fpm restart

	- Open nginx configuration file `/etc/nginx/sites-available/yoursite` and replace the line block starting with `location ~ \.php` with:
				
				location ~ \.php$ {
	                try_files $uri =404;
	                fastcgi_pass unix:/var/run/php5-fpm.sock;
	                fastcgi_index index.php;
	                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	                include fastcgi_params;
	                }

	- In order to check if it is all OK, create a new file:
		
			sudo nano /var/www/html/info.php
	
		and put the following line:
	
			<?php phpinfo(); ?>

	Now restart nginx and check that the server respond on the request of the page `info.php`.
	You should remove that file, since it provides useful information for an attacker.
	
- MySQL configuration:

		sudo mysql_secure_installation
	
	


## Step 5: Obtain a Digital Certificate

### Certbot

[Certbot](https://certbot.eff.org/) is an easy-to-use automatic client that fetches and deploys SSL/TLS certificates for your web server.
We will use it for manage our new digital certificate, that we can use in many ways.

First of all, you need to add [backports](https://backports.debian.org/Instructions/) to your `/etc/apt/sources.list` file.
Hence, open that file:

    sudo nano /etc/apt/sources.list

And add this line at the end of the file:
    
	deb http://ftp.debian.org/debian jessie-backports main

	
	sudo apt-get update
	

Now you can install it:

	sudo apt-get install certbot -t jessie-backports	
	
Then, run:

	sudo certbot certonly


In order to renew the certificate, run:

	sudo certbot renew
	
[Here](https://certbot.eff.org/docs/using.html#renewal) you will find details for the automatic renew.
Now you can use the new `.pem` files e.g. for make the server establish HTTPS connections.

### Integrate the digital certificate in your web server
We will use `nginx` as before.
For furhter details, we redirect [here](https://certbot.eff.org/docs/using.html#nginx).
We got inspiration from [this guide](https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-nginx-for-ubuntu-14-04).

Check that you have your certificates:

	sudo ls -l /etc/letsencrypt/live/<your-domain-name>/
	
	
Generate strong Diffie-Hellman Group. It takes a lot on a normal computer, an eternity on a little Raspberry Pi: I suggest you to run it on your computer and then copy the file to your server.

	sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048

Then, edit the following Nginx configuration file:

	sudo nano sites-available/default

Adding/uncommenting the following lines:

	listen 443 ssl;

	server_name example.com www.example.com;

	ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

To allow only the most secure protocols and ciphers, add the gollowing lines:

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;

Next, redirect HTTP requests to HTTPS requests:

	server {
	    listen 80;
	    server_name example.com www.example.com;
	    return 301 https://$host$request_uri;
	}

Test if there is some misconfiguration
  
  	sudo nginx -t

If it is all ok, restart Nginx server:

	sudo service nginx restart

If you want, you can test your security typing this URL in your browser:

	https://www.ssllabs.com/ssltest/analyze.html?d=example.com

You should get an **A+**.

## Keep your server updated

Updating your server

It's very important to keep you server updated. I get notification by email about new updates and perform them manually.

To receive these notifications, get Apticron:

	sudo apt-get install apticron

Enter your email address in the configuration file:

	sudo nano /etc/apticron/apticron.conf

Save and exit the file.

Now you will receive emails with update notifications.

To perform an update, just type:

	sudo apt-get dist-upgrade


## Miscellanea
### Configure locale settings
It may happen that your locale are misconfigured: follow [this guide](https://askubuntu.com/questions/162391/how-do-i-fix-my-locale-issue) for solve it.
In my case, it worked:

	$ sudo locale-gen "en_US.UTF-8"
	Generating locales...
	  en_US.UTF-8... done
	Generation complete.

	$ sudo dpkg-reconfigure locales
	Generating locales...
	  en_US.UTF-8... up-to-date
	Generation complete.
	
### Disable unused services
Installing `rcconf`, you can easily see all the available services and start/stop as you wish.

### Install additional software
- iptables-persistent

		sudo apt-get install iptables-persistent

- Postfix

		sudo apt-get install mailutils

- SFTP
	Once you configured SSH, you can easily use `sftp` as the following:
	
		sftp <username>@<host>
		
	Like you're logging with SSH.

- Python
Install `python3`:

		sudo apt-get install python3
		
	Install `pip`:
	
		wget https://bootstrap.pypa.io/get-pip.py
		sudo python3 get-pip.py
	
	For upgrade `pip`:
		
		sudo pip install -U pip
		
### Useful commands
Some useful commands in terminal

Updating the operating system

	sudo apt-get update && sudo apt-get upgrade

Updating the firmware

	sudo rpi-update

Secure copy local file to Pi

	scp -r local-path username@IPADDRESS:remote-path

Secure copy remote file to local directory

	scp -r username@IPADDRESS:remote-path local-path

Create bit by bit backup copy of SD Card

	sudo dd if=/dev/mmcblk0 of=/data/backups/raspberryPiSDCardBackup.img

Copy backup file from pi to local backup folder

	scp -r username@IPADDRESS:/data/backups/raspberryPiSDCardBackup.img /users/username/documents/projects/backups

Create SD Card with backup image:

1. Go to terminal on Mac

2. Insert Empty Formatted SD Card

3. Run (use your own username, ipaddress, directory and diskname):

		dd if=username@IPADDRESS:/data/backups/raspberryPiSDCardBackup.img of=/dev/disk2s1



## References

- [What should be done to secure Raspberry Pi?](https://raspberrypi.stackexchange.com/questions/1247/what-should-be-done-to-secure-raspberry-pi) 
- [Secure your server](https://www.linode.com/docs/security/securing-your-server/)
- [Comprehensive guide on the same topic](https://www.pestmeester.nl/)
- [Choose a secure password](http://www.wikihow.com/Create-a-Secure-Password)
- [SSH configuration](https://help.ubuntu.com/community/SSH/OpenSSH/Configuring)
- [SSH Public/Private keys](https://help.ubuntu.com/community/SSH/OpenSSH/Keys#Password_Authentication)
- [Removing Unwanted Startup Debian Files or Services](https://theos.in/desktop-linux/removing-unwanted-startup-debian-files-or-services/)
- [fail2ban Wiki](http://www.fail2ban.org/wiki/index.php/Main_Page)
- [Tutorial on fail2ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [How fail2ban works](https://www.digitalocean.com/community/tutorials/how-fail2ban-works-to-protect-services-on-a-linux-server)
- [Debianizzati tutorial on fail2ban (italian)](http://guide.debianizzati.org/index.php/Fail2ban)
- [fail2ban for Apache](https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04)
- [fail2ban for Nginx](https://www.digitalocean.com/community/tutorials/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-14-04)
- [How To Set Up a Firewall Using Iptables](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-iptables-on-ubuntu-14-04)
- [CertBot homepage](https://certbot.eff.org/)
