# Linux Server Configuration

The aim of this project is to gain hands-on experience configuring and securing a linux server to deploy a web application. Amazon lightsail was used to setup a virtual machine instance.


### Server Details

IP Address: `13.126.115.120`

SSH Port: `2200`

URL to access application: [http://ec2-13-126-115-120.ap-south-1.compute.amazonaws.com/](http://ec2-13-126-115-120.ap-south-1.compute.amazonaws.com/)


### Server Configuration

1. Update all currently installed packages

```
sudo apt-get update
sudo apt-get upgrade
```

Use command `sudo reboot` in case system restart is required after upgrade.

2. Change SSH port from the default port `22` to `2200`

	Edit file `sshd_config`

	```
	sudo vi /etc/ssh/sshd_config
	```

	* Change the line `Port 22` to `Port 2200`

	* Restart ssh service

	```
	sudo service ssh restart
	```

3. Configure Uncomplicated Firewall (UFW)

	* By default, block incoming connections on all ports

	```
	sudo ufw default deny incoming
	```

	* Allow outgoing connections on all ports

	```
	sudo ufw default allow outgoing
	```

	* Allow incoming connections for 

		a. SSH (`Port 2200`)

		```
		sudo ufw allow 2200/tcp
		```

		b. HTTP (`Port 80`)

		```
		sudo ufw allow http
		```

		c. NTP (`Port 123`)

		```
		sudo ufw allow ntp
		```

	* Enable firewall

	```
	sudo ufw enable
	```

	* Check the status of firewall using

	```
	sudo ufw status
	```

4. Setup user account for `grader`

	* Create new user named `grader`

	```
	sudo adduser grader
	```
	Enter a password for user `grader` when prompted.

	* Give grader permission to `sudo`
	Create a new file named `grader` in `sudoers.d` directory
	
	```
		sudo vi /etc/sudoers.d/grader
	```
	Add the following line in the new file
	```
	grader ALL=(ALL:ALL) ALL
	```

	* Create SSH key-pair for grader
	
	```
	su - grader
	ssh-keygen

	Generating public/private rsa key pair.
	Enter file in which to save the key (/home/grader/.ssh/id_rsa):
	Created directory '/home/grader/.ssh'.
	Enter passphrase (empty for no passphrase):
	Enter same passphrase again:
	Your identification has been saved in /home/grader/.ssh/id_rsa.
	Your public key has been saved in /home/grader/.ssh/id_rsa.pub.
	```

	* Disable Password Authentication and remote login of `root` user

	Edit file `/etc/ssh/sshd_config` and set the following lines
	```
	PasswordAuthentication no
	```
	and

	```
	PermitRootLogin no
	```

	Restart ssh service for changes to take effect

	```
	sudo service ssh restart
	```

	* User `grader` can use the following command to login with the SSH key provided
	
	```
	ssh -i grader_key grader@13.126.115.120 -p 2200
	```

5. Prepare server to deploy web application

	* Configure local timezone to UTC

	```
	sudo dpkg-reconfigure tzdata
	```
	Select `None of the above` under geographic area menu and `UTC` in timezone menu



	* Intall NTP Daemon to continously sync the system clock

	```
	sudo apt-get install ntp
	```

	* Install Apache

	```
	sudo apt-get install apache2
	```

	* Install mod_wsgi package

	```
	sudo apt-get install libapache2-mod-wsgi
	```

	* Install python-pip
	
	```
	sudo apt-get intall python-pip
	```

	* Install required python packages.

	```
	sudo pip install -r requirements
	```

	* Install PostgreSQL

	```
	sudo apt-get install postgresql
	```

	Verify remote connection setting in the file `/etc/postgres/9.5/main/pg_hba.conf` and confirm only connections from localhost is allowed.
	`127.0.0.1 for IPv4 and ::1 for IPv6`.

	* Configure PostgreSQL

	Login as `postgres` user and open PostgreSQL interactive terminal

	```
	sudo su - postgres
	psql
	```

	Create database and user for Catalog Application
	```
	postgres=# CREATE DATABASE catalog;
	postgres=# CREATE USER catalog;
	```

	Set password for catalog user

	```
	postgres=# ALTER ROLE catalog WITH PASSWORD 'catalogdba';
	```

	Grant all privileges to catalog user on catalog database

	```
	postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog to catalog;
	```

	Exit from PostgreSQL terminal and exit postgres user

	```
	postgres=# \q

	exit
	```


	* Install GIT

	```
	sudo apt-get install git
	```

	* Deploy and setup catalog application

	Change current directory to `/var/www`

	```
	cd /var/www
	```

	Clone the CatalogApp project using following command

	```
	git clone https://github.com/arunphilip03/CatalogApp.git
	```


	Make following changes to the Catalog Application code `cd CatalogApp`

	* Rename `catalogApp.py` to `__init__.py`
	* Edit database_setup.py, insert_data.py and __init__.py to change database to postgres instead of sqllite

	```
	#engine = create_engine('sqlite:///itemcatalog.db')
	```
	changed to 
	
	```
	engine = create_engine('postgresql://catalog:catalogdba@localhost:5432/catalog')
	```

	Create database schema
	
	```
	sudo python database_setup.py
	```

	Load initial data into catalog database

	```
	sudo python insert_data.py
	```

	#### Configure Apache to serve the web application

	Create virtual host configuration file
	Create file `catalogApp.conf` in `/etc/apache2/sites-available`

	```
	sudo vi /etc/apache2/sites-available/catalogApp.conf
	```

	Add the following configuration in the new file:

	```xml
	<VirtualHost *:80>
        ServerAdmin philip03@gmail.com

        # Define the location of the app's WSGI file
        WSGIScriptAlias / /var/www/CatalogApp/catalogapp.wsgi

        # Allow Apache to serve the WSGI app from the Catalog App directory
        <Directory /var/www/CatalogApp/>
                Require all granted
        </Directory>

        # Setup the static directory
        Alias /static /var/www/CatalogApp/static

        # Allow Apache to serve the files from the static directory
        <Directory  /var/www/CatalogApp/static/>
                Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
	</VirtualHost>

	```

	Disable the default virtual host configuration

	```
	sudo a2dissite 000-default.conf
	```

	Enable the `catalogApp.conf`

	```
	sudo a2ensite catalogApp
	```

	Reload apache service for changes to take effect

	```
	sudo service apache2 reload
	```

	Create wsgi file in `/var/www/CatalogApp` directory to enable apache web server to invoke the catalog application

	+ Create file `catalogapp.wsgi`
		
	```
	cd /var/www/CatalogApp
	sudo vi catalogapp.wsgi
	```

	+ Add the following code in the file

	```python
	#!/usr/bin/python

	import sys
	import logging
	logging.basicConfig(stream=sys.stderr)
	sys.path.insert(0, '/var/www/CatalogApp')

	from catalog import app as application

	application.secret_key = 'super_secret_key'

	```

	### Google and Facebook OAuth changes

	Following changes were made to enable login using either Google or Facebook.

	+ Add the following Authorized Javascript origins in Google developers console -> API & Services -> Credentials -> Catalog Application

	```
	http://ec2-13-126-115-120.ap-south-1.compute.amazonaws.com
	http://13.126.115.120
	```

	+ In Facebook developers console -> Settings -> Basic, the Site URL under WebSite section was changed to

	```
	http://ec2-13-126-115-120.ap-south-1.compute.amazonaws.com
	```


	### About Catalog Application

	Catalog App is a web application that provides a list of items within a variety of categories. It integrates with third party providers for authentication and user registration. Currently this project supports sign-in using either Google or Facebook accounts. Once signed in, users will have the ability to Add, Edit and Delete their own items.

















