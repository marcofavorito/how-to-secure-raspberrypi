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
        
And typing the new password. Please, [choose a strong password... and remember it!!!](https://www.howtogeek.com/195430/how-to-create-a-strong-password-and-remember-it/)

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
In this guide, I will use [No-IP](https://www.noip.com/).
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

## Step 4: Install a Web Server
- Nginx

		sudo apt-get install nginx
		
	Useful commands:
	
		sudo systemctl (disable|enable) nginx.service
		sudo service nginx (start|stop|restart)

	Configuration files in `/etc/nginx/`.
	
## Step 5: Obtain a Digital Certificate
[CertBot](https://certbot.eff.org/) is an easy-to-use automatic client that fetches and deploys SSL/TLS certificates for your webserver.
We will use it for manage our new digital certificate, that we can use in many ways.

First of all, you need to add [backports](https://backports.debian.org/Instructions/) to your `sources.list` file.
	
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

## Miscellanea
### Disable unused services
Installing `rcconf`, you can easily see all the available services and start/stop as you wish.

### Install additional software
- cron-apt

		sudo apt-get install cron-apt

- iptables-persistent

		sudo apt-get install iptables-persistent

- Postfix

		sudo apt-get install mailutils

- SFTP
	Once you configured SSH, you can easily use `sftp` as the following:
	
		sftp <username>@<host>
		
	Like you're logging with SSH.

## References

- [What should be done to secure Raspberry Pi?](https://raspberrypi.stackexchange.com/questions/1247/what-should-be-done-to-secure-raspberry-pi) 
- [Choose a secure password](http://www.wikihow.com/Create-a-Secure-Password)
- [SSH configuration](https://help.ubuntu.com/community/SSH/OpenSSH/Configuring)
- [SSH Public/Private keys](https://help.ubuntu.com/community/SSH/OpenSSH/Keys#Password_Authentication)
- [Removing Unwanted Startup Debian Files or Services](https://theos.in/desktop-linux/removing-unwanted-startup-debian-files-or-services/)
- [fail2ban Wiki](http://www.fail2ban.org/wiki/index.php/Main_Page)
- [Tutorial on fail2ban](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [Debianizzati tutorial on fail2ban (italian)](http://guide.debianizzati.org/index.php/Fail2ban)
- [fail2ban for Apache](https://www.digitalocean.com/community/tutorials/how-to-protect-an-apache-server-with-fail2ban-on-ubuntu-14-04)
- [fail2ban for Nginx](https://www.digitalocean.com/community/tutorials/how-to-protect-an-nginx-server-with-fail2ban-on-ubuntu-14-04)
- [How To Set Up a Firewall Using Iptables](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-iptables-on-ubuntu-14-04)
- [CertBot homepage](https://certbot.eff.org/)