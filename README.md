# Udacity Linux Server Configuration 

This is the final project for Udacity's [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004). 

Using baseline installation of a Linux distribution on a virtual machine prepare it to host BookCatalog  web application including installing updates, securing  from a number of attack vectors and installing/configuring web and database servers.

- The Linux distribution is [Ubuntu](https://www.ubuntu.com/download/server) 16.04 LTS.
- The virtual private server is [Amazon Lighsail](https://lightsail.aws.amazon.com/).
- Application URL: http://35.176.103.212.xip.io/
- Application IP: 35.176.103.2102
- Application name: BookCatalog
- Port for SSH connection 2200
-	ssh -i ~/.ssh/ privateKey grader@ 35.176.103.212
- 	ssh -i [key_file_path] grader@52.42.38.177 -p 2200

## Start a new Ubuntu Linux server instance on Amazon Lightsail 
- ServerPilot, [How to Create a Server on Amazon Lightsail](https://serverpilot.io/community/articles/how-to-create-a-server-on-amazon-lightsail.html).
- Udacity [Get started on lightsail](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e)
## Configuration 

### Update all installed packages + automatic update 

- sudo apt-get update  				- *update the package source list*_ 
- sudo apt-get upgrade 				- *upgrade the packages*

**Automatic updates**
- sudo apt-get install unattended-upgrades - *enable automatic updates*
- sudo nano /etc/apt/apt.conf.d/50unattended-upgrades 
	1. find line and un-comment : "${distro_id}:${distro_codename}-updates"
	3. save it
- sudo nano /etc/apt/apt.conf.d/20auto-upgrade
    ```
    APT::Periodic::Update-Package-Lists "1";
    APT::Periodic::Download-Upgradeable-Packages "1";
    APT::Periodic::AutocleanInterval "7";
    APT::Periodic::Unattended-Upgrade "1"
    ```
- sudo dpkg-reconfigure --priority=low unattended-upgrades - *enable*

### Change the SSH port from 22 to 2200
- vi /etc/ssh/sshd_config			- *edit sshd config  file to change port number to 2200*
- service sshd restart 				- *restart the sshd service*

### Configure the UFW firewall 
The UFW will allow only incoming connection for SSH(Port 2200) , HTTP (Port 80), NTP (Port 123)
- sudo ufw default deny incoming 	- *block all incoming*
- sudo ufw default allow outgoing 	- *allow all outgoing*
- udo ufw allow 2200/tcp			- *allow incoming SSH connection*
- sudo ufw allow www				- *allow HTTP connection*
- sudo ufw ntp						- *allow NTP connection*
- sudo ufw enable					- *start the firewall*

### Protection against attacker using `Fail2Ban`
- sudo apt-get install fail2ban		- *Install Fail2Ban*
- sudo apt-get install sendmail iptables-persistent		- *email notice*
- sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local		- *Create a copy of a file*
- sudo nano /etc/fail2ban/jail.local	
	- `set bantime = 500`
	- `destemail = usermail@domain`
	- `action = %(acton_mwl)s`
- 	Under [sshd] change port = ssh by port = 2200.	- *change port to 2200*
- 	sudo service fail2ban restart	- *restart*

### Create additional user with sudo permission 
- sudo adduser grader			- *create use user named grader (password: grader)*
- sudo usermod -aG sudo grader 	- *give grader sudo privileges*

### Create an SSH pair for grader using ssh-keygen tool
Create SSH key Pair using ssh-keygen on local machine. This operation created two files 
graderKey. and graderKey.pub
	
Log as a grader user 
- mkdir .ssh  						- *create new directory*
- touch .ssh/authorized_keys  		- *create a file to store the server key*
- nano .ssh/authorized_keys			- *edit the file and past the content of the graderKey*
- chmod 700 .ssh					- *set directory permission*
- chmod 644  .ssh/authorized_keys	- *set file permission*	

Disable password login forcing  to use SSH  key pairs
- sudo nano /etc/ssh/sshd_config	- *change PasswordAuthentiation to No*
- sudo service ssh restart			- *Restart SSH service*

to Log in as a grader use : <br/>
	**ssh -i ~/.ssh/ privateKey grader@ 35.176.103.212**

### Configure the local time to UTC
- sudo timedatectl set-timezone Etc/UTC

### Install Apache and   mod_wsgi application
- sudo apt-get install apache2  			- *install apache*
- sudo apt-get install libapache2-mod-wsgi  - *install wsgi*

### Install Python, Flask  and all required libraries 
- sudo apt-get install python-psycopg2 python-flask
- sudo apt-get install python-sqlalchemy python-pip
- sudo pip install oauth2client
- sudo pip install requests
- sudo pip install httplib2
- sudo pip install flask-seasurf
- sudo apt-get install git
### Get application from git hub 
- cd  var/www/
- mkdir catalog
- sudo git clone https://github.com/Mtheory/BookCatalog.git
- the path to app is:  /var/www/catalog/BookCatalog
	
### Install and configure PostgreSQL
- sudo apt-get install postgresql - *install PostgreSQL*
- sudo vim /etc/postgresql/9.3/main/pg_hba.conf - *no remote connections are allowed*
- Create the catalog user with password:catalog and give them ability to create databases:
- Create a new database named catalog and create a new user named catalog in postgreSQL shell
- change the engine creation in project, database_setup and populate_database files to :<br/>
	`engine = create_engine('postgresql://user:pass@host:port/db_name)`

### Apache and wsgi configuration
- sudo nano /etc/apache2/sites-available/BookCatalog.conf -*Create a virtual host config file* <br/>
```
<VirtualHost *:80>
        ServerName 35.176.103.212.xip.io
        ServerAdmin edoforti@gmail.com
        WSGIScriptAlias / /var/www/catalog/BookCatalog/bookCatalog.wsgi
        <Directory /var/www/catalog/BookCatalog>
                Order allow,deny
                Allow from all
        </Directory>
        Alias /static /var/www/catalog/BookCatalog/static
        <Directory /var/www/catalog/BookCatalog/static>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/error.log combined
</VirtualHost>
```
- sudo a2ensite BookCatalog - *enable the BookCatalog app*
- sudo nano /var/www/catalog/BookCatalog/bookCatalog.wsgi - *create wsgi file*<br/>

```#! /usr/bin/python
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/BookCatalog")

from project import app as application

application.secret_key = '123Gvt45Ctd6lerT59'
```
- sudo service apache2 restart

### Helpful Links
* [Hello Flask with Apache WSGI](http://www.bogotobogo.com/python/Flask/Python_Flask_HelloWorld_App_with_Apache_WSGI_Ubuntu14.php)
* [Installing Apache](https://www.digitalocean.com/community/tutorials/apache-basics-installation-and-configuration-troubleshooting) 
* [Deploying Flask App on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-14-04
* https://www.digitalocean.com/community/tutorials/how-to-deploy-python-wsgi-applications-using-uwsgi-web-server-with-nginx
* [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* GitHub Repositories 
  1. [adityamehra/udacity-linux-server-configuration](https://github.com/adityamehra/udacity-linux-server-configuration)
  2. [iliketomatoes/linux_server_configuration](https://github.com/iliketomatoes/linux_server_configuration)



