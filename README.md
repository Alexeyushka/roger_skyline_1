# roger-skyline-1
This project, roger-skyline-1 let you install a Virtual Machine, discover the basics about system and network administration as well as a lots of services used on a server machine.

## Summary <a id="summary"></a>

- [Summary](#summary)
- [Virtual Machine Installation](#VMinstall)
- [Set up the Environment](#SetUpEnvironment)
- [Web Part](#WebPart)


## Part 1. Virtual Machine Installation <a id="VMinstall"></a>

1. [Download the official Debian image (9.6.0)](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-9.6.0-amd64-netinst.iso)
2. [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)
3. Create a debian 8gb fixed size machine with VirtualBox
4. Install the previously downloaded image of debian
5. Before launching the virtual machine. In VirtualBox, go to the host's network manager (files-> Host Network Manager). Click Create, and then uncheck the DHCP server. You should have a vboxnet0 host with the ip 192.168.56.1/24.
6. Go to the settings of the previously created Debian machine: Settings-> Network. For card 1, leave access mode NAT, then enable card 2 in "Private Host Network" mode. It will default to vboxnet0.
7. Start the VM
8. Create a 4.2Gb partition (in creation, create one of 4.501gb) with / in mountpoint. Then a partition of 1gb type /swap and finally with the rest of the hard disk, make a partition /home.
9. Install ssh and usual services

## Part 2. Setting up the Environment <a id="SetUpEnvironment"></a>

1. Install the necessary packages for Roger-Skyline:
```bash
apt install -y sudo net-tools iptables-persist fail2ban sendmail apache2
```

2. Then modify the ssh parameters:
```bash
vim /etc/ssh/sshd_config
```
- Change the port to 2222 (for example)
- Uncomment the #PasswordAuthentification yes line
- Uncomment #PermitRootLogin prohibit-password and replace with No
- Uncomment #PubkeyAuthentication yes
3. Now create a new network interface that will link your machine to your host:
```bash
vim /etc/network/interfaces
```
And add the following content:

```bash
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces (5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
address 192.168.56.3
netmask 255.255.255.252
```
4. Create a new user now:
```bash
adduser <username>
```
I used username `newuser`
5. Enter the info and add it to the sudo group:
```bash
adduser <username> sudo
```
6. Restart the machine now:
```bash
reboot
```
7. In a terminal of your host, generate a public ssh key:
```bash
ssh-keygen
```
8. Cat and copy the content
```bash
cat ~/.ssh/id_rsa.pub
```
9. Connect now to your machine:
```bash
ssh <user>@<ip machine> -p <Port>
```
in our case use command:
```bash
ssh newuser@127.0.0.1 -p 2222
```
10. Enter the password, required first time
11. Create .ssh folder in your user's home:
```bash
mkdir .ssh
```
12. Open .ssh/authorized_keys and paste the key into the file
```bash
vim .ssh/authorized_keys
```
13. Edit the ssh configuration file (change "PasswordAuthentification "yes" to "no":
```bash
sudo vim /etc/ssh/sshd_config
```
14. Restart the ssh service
```bash
sudo service ssh restart
```
15. Exit the ssh with exit and retry a connection with the previous command. The connection is now password-free with PublicKeys. You will find your VM in the .ssh / known_hosts file.

## Setup Firewall with UFW. <a id="ufw"></a>

1. Make sure ufw is enable

```bash
sudo ufw status
```
 if not we can start the service with
 
 ```bash
 sudo ufw enable
 ```
 
2. Setup firewall rules
      - SSH : `sudo ufw allow 50683/tcp`
      - HTTP : `sudo ufw allow 80/tcp`
      - HTTPS : `sudo ufw allow 443`
      
3. Setup Denial Of Service Attack with fail2ban
      
```bash
sudo vim /etc/fail2ban/jail.conf
```

```console
[sshd]
enabled = true
port    = 42
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 600

#Add after HTTP servers:
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/apache2/access.log (le fichier d'access sur server web)
maxretry = 300
findtime = 300
bantime = 600
action = iptables[name=HTTP, port=http, protocol=tcp]
```

Add http-get-dos filter

```bash
sudo cat /etc/fail2ban/filter.d/http-get-dos.conf
```

Output:

```console
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*
ignoreregex =
```
5. Finally we need to reload our firewall and fail2ban

```bash
sudo ufw reload
sudo service fail2ban restart
```

## Protection against port scans. <a id="scanSecure"></a>

1. Config portsentry

First, we have to edit the `/etc/default/portsentry` file

```console
TCP_MODE="atcp"
UDP_MODE="audp"
```

After, edit the file `/etc/portsentry/portsentry.conf`

```console
BLOCK_UDP="1"
BLOCK_TCP="1"
```

Comment the current KILL_ROUTE and uncomment the following one:

```console
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```

Comment the following line:

```console
KILL_HOSTS_DENY="ALL: $TARGET$ : DENY
```

2. We can now restart the service to make changes effectives

```bash
sudo service portsentry restart
```

## Stop the services we don’t need <a id="stopServices"></a>

```bash
sudo systemctl disable console-setup.service
sudo systemctl disable keyboard-setup.service
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl disable syslog.service
```

## Update Packages <a id="updateApt"></a>

1. Create the `update.sh` file and write the following lines inside

```bash
echo "sudo apt-get update -y >> /var/log/update_script.log" >> ~/update.sh
echo "sudo apt-get upgrade -y >> /var/log/update_script.log" >> ~/update.sh
```

2. Add the task to cron

```bash
sudo crontab -e
```

3. Write in the openned file thoses lines

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

@reboot sudo ~/update.sh
0 4 * * 6 sudo ~/update.sh
```

## Monitor Crontab Changes <a id="cronChange"></a>

1. Create the `cronMonitor.sh` file and write the following lines inside

```console
gde@roger-skyline-1:~$ cat ~/cronMonitor.sh
#!/bin/bash

FILE="/var/tmp/checksum"
FILE_TO_WATCH="/var/spool/cron/crontabs/gde"
MD5VALUE=$(sudo md5sum $FILE_TO_WATCH)

if [ ! -f $FILE ]
then
	 echo "$MD5VALUE" > $FILE
	 exit 0;
fi;

if [ "$MD5VALUE" != "$(cat $FILE)" ];
	then
	echo "$MD5VALUE" > $FILE
	echo "$FILE_TO_WATCH has been modified ! '*_*" | mail -s "$FILE_TO_WATCH modified !" root
fi;
```

2. Add the task to cron

```bash
crontab -e
```

3. Write in the openned file thoses lines

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

@reboot sudo ~/update.sh
0 4 * * 6 sudo ~/update.sh
0 0 * * * sudo ~/cronMonitor.sh
```

4. Make sure we have correct rights

```bash
sudo chmod 755 cronMonitor.sh
sudo chmod 755 update.sh
sudo chown newuser /var/mail/newuser
```

5. make sure cron service is enable

```bash
sudo systemctl enable cron
```


## Web Part <a id="WebPart"></a>
1. Generate a new SSL key, enter the info when requested.:
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -subj "/C=FR/ST=IDF/O=42/OU=Project-roger/CN=10.11.200.247" -keyout /etc/ssl/private/roger-skyline.com.key -out /etc/ssl/certs/roger-skyline.com.crt
```

2. Then
```bash
sudo vim /etc/apache2/sites-available/default-ssl.conf
```
SSL, indicate the correct way of the keys:
```bash
<IfModule mod_ssl.c>
<VirtualHost _default_: 443>
Webmaster ServerAdmin @ localhost
DocumentRoot /var/www/html

ErrorLog $ {APACHE_LOG_DIR} /error.log
CustomLog $ {APACHE_LOG_DIR} /access.log combined

#Include conf-available / serve-cgi-bin.conf

# SSL engine switch:
# Enable / Disable SSL for this virtual host.
SSLEngine on
SSLCertificateFile /etc/ssl/certs/roger-skyline.com.crt
SSLCertificateKeyFile /etc/ssl/private/roger-skyline.com.key
#
#SSLCertificateFile /etc/ssl/certs/ssl-cert-snakeoil.pem
#SSLCertificateKeyFile /etc/ssl/private/ssl-cert-snakeoil.key

......................
.......................

</ VirtualHost>
</ IfModule>
```
3. Test the following commands:
```bash
sudo apachectl configtest
sudo a2enmod ssl
sudo a2ensite default-ssl
```
if Error occurs:
```bash
sudo systemctl restart apache2.service
```

4. Then make a copy of the default configuration:
```bash
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/001-default.conf
```

5. And modify the file south 
```bash
vim /etc/apache2/sites-available/001-default.conf
```
Change the ServerName by whatever you want and the DocumentRoot by the path to your website.
Activate the new configuration:

6. Disables the old configuration
```bash
a2dissite 000-default.conf
```
7. Active new
```bash
a2ensite 001-site.conf
```
8. The command speaks for itself ...
```bash
systemctl reload apache2
```

The site will normally be accessible on your IP (https://192.168.56.3).
This is a self signed certificate so the browser will warn you before accessing, just skip it.

6. You can add files in the folder if you have not changed the DocumentRoot.
```bash
/var/www/html
```
