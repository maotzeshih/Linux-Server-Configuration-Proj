Project2:Project:Linux Server Configuration
===========================================
## Project Description
Take a baseline installation of a Linux distribution on a virtual machine and prepare it to host a web applications, 
to include installing updates, securing it from a number of attack vectors and installing/configuring web and 
database servers.
Public IP: 18.207.2115.119
HTTP port: 80 (default)
SSH port: 2200
Application URL: http://www.maotze.com or http://www.maotze.com/catalog

## Setup ubuntu server environment
1. Download private keys and write down remote IP address (18.207.2115.119) from aws lightsail account
2. Convert private key using PuTTYgen from (.pem) to (.ppk) format
3. Connect remote aws linux instance via PuTTY Configuration
 
## Create new user
1. Add new user 
$ sudo adduser grader (password: newcomer)
2. Add grader for all root privileges 
$ sudo vim /etc/sudoers.d/grader (add this line --> grader ALL=(ALL) ALL)
Note: Confirmed that grader has root privileges by running the following commands
$ sudo su - grader
$ sudo whoami (it should return root) 

## Update and upgrade all currently installed packages
$ sudo apt-get update
$ sudo apt-get upgrade

## Secure ubuntu server
1. Change SSH port 
$ sudo vim /etc/ssh/sshd_config (change from 22 to 2200) 
2. Open port 2200 for firewall
Note: The firewall and port settings are listed in the Networking tab of instance's management page in Lightsail console. 
Choose "Add another" to create rule for 2200.
      a. Go to Go to Amazon Lightsale
      b. Click Networking tab
      c. Add the following line:
         application  Protocol   Port range
         Custome      TCP        2200
3. copy key-based authentication from ubuntu to grader 
$ cp /home/ubuntu/.ssh/authorized_keys /home/grader/.ssh/authorized_keys
4. Connect server as grader using port 2200 via PuTTY Configuration
Note: Confirmed that the Lightsail instance is no longer accessible through the web app 'Connect using SSH' button. 
      a. Go to Amazon Lightsale
      b. Click "Connect using SSH" button
5. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
$ sudo ufw allow 2200/tcp  
$ sudo ufw allow 80/tcp    
$ sudo ufw allow 123/udp   
$ sudo ufw enable        

## Pre-requisites
1. Configure local time to utc 
$ sudo dpkg-reconfigure tzdata (select UTC from the list)
2. Install Apache web server
$ sudo apt-get install apache2
3. Install mod_wsgi to enable apache to serve flask app
$ sudo apt-get install libapache2-mod-wsgi python-dev
4. Enable mod_wsgi
$ sudo a2enmod wsgi
5. Restart Apache 
$ sudo service apache2 reload

## Installation
1. Clone catalog web application from Github:
$ sudo apt-get install git
$ cd /var/www
$ sudo mkdir catalog
$ cd catalog
$ sudo git clone https://github.com/maotzeshih/catalog_app.git catalog
2. Create /var/www/catalog/catalog.wsgi 
$ cd /var/www/catalog
$ sudo vim catalog.wsgi

    import sys
    sys.stdout = sys.stderr
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")
    from catalog import app as application
    application.secret_key = 'supersecretkey'

3. Change catalog.wsgi permission and then restart Apache
$ chmod 777 catalog.wsgi
$ sudo service apache2 restart
4. Rename application name to __init__.py
sudo mv ma2.py __init__.py
5. Setup virtual environment
$ cd /var/www/catalog
$ sudo -H pip install virtualenv
$ sudo virtualenv venv
$ source venv/bin/activate
6. Change permissions
$ sudo chmod -R 777 venv

## Dependencies/Pre-requisites
1. catalog application is using python version 2.7.15+, sqlalchemy version 1.3.3 and flask version 1.03
Additional packages required for this project are: marshmallow version 3.0.0rc5 and marshmallow-sqlalchemy version 0.16.3
Use pip command in the shell to install the following required packages
$ sudo pip install sqlalchemy
$ sudo pip install Flask
$ sudo pip install flask-sqlclchemy
$ sudo pip install flask_marshmallow
$ sudo pip install marshmallow
$ sudo pip install marshmallow-sqlalchemy
$ sudo pip install httplib2
$ sudo pip install oauth2client
$ sudo pip install requests

## Setup virtual host
1. Make sure apache2 listening port 80
$ netstat -lntp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN
2. Create configuration file so Apache can handle requests using the WSGI module
$ sudo vim /etc/apache2/sites-available/catalog.conf

<VirtualHost *:80>
    ServerName 18.207.115.119
    ServerAdmin admin@18.207.115.119
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
		<Directory /var/www/catalog/catalog/>
			Order allow,deny
			Allow from all
			Options -Indexes
		</Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

3. Enable virtual host
$ sudo a2ensite catalog
4. Set up database and import data for user, category and gear_item so the database gearswithuses.db is ready for use
$ cd /var/www/catalog/catalot
$ sudo python database_setup.py
$ sudo python lotsofitems.py
4. Restart apache
$ sudo service apache2 reload

## Authenticate user using Google sign-in 
1. Go to google domains to create a new records for domain wwww.maotze.com pointing it at ip address 18.207.115.119
   a. Type https://domains.google.com/m/registrar/maotze.com/dns in browser
   b. Scroll down to "Custom resource records"
   c. Add the following line
   Name    Type   TTL     data
   www     A      1h      18.207.115.119

2. Go to the Google API Console to create a client ID and client secret 
   a. Add http://18.207.115.119 and http://www.maotze.com as authorized JavaScript origins
   b. Add http://www.maotze.com/login, http://www.maotze.com/gconnect and http://www.maotze.com as as authorized redirect URI
   c. Download JSON file and copy contents to /var/www/catalog/catalog/client_secret.json
   d. Replace client id in /var/www/catalog/catalog/templates/login.html

## Usage  
1. To run catalog app, enter the following URL:
http://www.maotze.com and then click "catalog" link or http://www.maotze.com/catalog


## Debugging & Known Issues
1. Check apache2 firewall
$ sudo ufw status
Status: active
To                         Action      From
--                         ------      ----
2200/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123/udp                    ALLOW       Anywhere
Apache                     ALLOW       Anywhere
2200/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123/udp (v6)               ALLOW       Anywhere (v6)
Apache (v6)                ALLOW       Anywhere (v6)

2. Check if apache is running
$ sudo systemctl status apache2

3. Check error.log
$ cd /var/log
$ sudo tail -f apache2/error.log

a. fix OperationalError: (sqlite3.OperationalError) unable to open database file, referer: http://18.207.115.119/
(Background on this error at: http://sqlalche.me/e/e3q8), referer: http://18.207.115.119/
update data path --> engine = create_engine('sqlite:////var/www/catalog/catalog/gearswithusers.db')
b. fix threading issue --> engine = create_engine('sqlite://.....',
                    connect_args={'check_same_thread':False}
                    )
c. update client secrets file path to /var/www/catalog/catalog/client_secret.json
d. fix SQLALCHEMY_TRACK_MODIFICATIONS warning --> app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
e. fix SQLAlchemy configuration warning --> app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:////var/www/catalog/catalog/gearswithusers.db'


## Future Plans
None

## Contribution guidelines
None   

## License
This is free and unencumbered software released into the public domain
