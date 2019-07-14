# Linux Server Configuration

_A Udacity Fullstack Web Developer nanodegree project._

A baseline installation of a Linux server and prepare it to host my web applications.

APP: [Starship catalog app running](http://ec2-54-236-62-234.compute-1.amazonaws.com/starships/)

Public IP: 54.236.62.234 (RIP)

## Objectives of the project

[rubrick](https://review.udacity.com/#!/rubrics/7/view)

## Steps

### Create a server

1. get a Lightsail Ubuntu server and access it [according to Udacity Linux security Course - Get started on Lightsail](https://classroom.udacity.com/nanodegrees/nd004-br/parts/4cb4df39-2f34-4f6e-95ba-6b541da97160/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e)
2. Access the server and update its configurations
```shell
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
3. Configure timezone to UTC
```shell
sudo timedatectl set-timezone UTC
date
```
### Configure an user to work on the server

#### Create a new user named grader
```shell 
sudo adduser grader
#to create the user
sudo chage -d 0 grader
#to make grader change its pw at first login
sudo usermod -aG sudo grader
#to grant sudo to grader, add it to Sudoers
```

#### Set-up SSH keys for user grader

1. Configure keys on your local machine:
```shell
ssh-keygen grader-key
# Enter file name to save the key: grader
# Let's keep passphrase empty
cp grader-key ~/.ssh
#to save the private key in ~/.ssh on local machine
cat grader-key.pub 
#to copy it's content to paste at server
```

2. Deploy public key on developement enviroment

```shell
sudo mkdir /home/grader/.ssh
#to create a .ssh directory
sudo chown grader:grader /home/grader/.ssh
#to give ownership of the dir to grader
sudo touch /home/grader/.ssh/authorized_keys
#to create a authorized_keys file and edit it,
sudo nano /home/grader/.ssh/authorized_keys
#to add the key.pub value to it
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
#to give the needed permission to folder and files
```
**now you can ssh to login with the grader**

_I've paste the contents of the grader user's SSH key into the "Notes to Reviewer" field._

- ```ssh -i grader-key grader@54.236.62.234 -p 2200``` 

### Configure Security

#### Remove root user access

```shell
sudo nano /etc/ssh/sshd_config
```
- Change the following lines:
 1. FROM: ```PermitRootLogin without-password ``` TO: ```PermitRootLogin no``` 
 2. FROM: ```PasswordAuthentication yes``` TO: ```PasswordAuthentication no```
 3. ADD: ```DenyUsers root```

#### Change SSH port from 22 to 2200

```shell
sudo nano /etc/ssh/sshd_config
#to change Port from 22 to 2200
```

- Restart SSH service to active security changes above

```shell
sudo service ssh restart
```

#### Configure Firewall
```shell
sudo ufw status
sudo ufw default deny incoming
#to block all incoming connections on all ports by default
sudo ufw default allow outgoing
#to allow outgoing connection on all ports
sudo ufw allow 2200/tcp
#to allow incoming connection for SSH on port 2200
sudo ufw allow www 
#to allow incoming connections for HTTP on port 80
sudo ufw allow ntp
#to allow incoming connection for NTP on port 123
sudo ufw enable
#to enable the firewall, use
sudo ufw status
#to make sure everything is fine
```

### Deploy Catalog app on the Server

#### Setup Apache to serve a Python mod_wsgi application

```shell
sudo apt-get install apache2
#to install appache
sudo apt-get install python-setuptools libapache2-mod-wsgi
#to install Install mod_wsgi 
sudo service apache2 restart
#to restart apache
```

#### Setup PostgreSQL

```shell
sudo apt-get install postgresql
#to install appache
sudo nano /etc/postgresql/9.3/main/pg_hba.conf
#to check if no remote connections are allowed
sudo su - postgres
#to login as postgres
```

- as postgree get into postgreSQL shell with ```psql``` in there do:

1. Create a new database named catalog
```
postgres=# CREATE DATABASE catalog;
```

2. Create a new user named catalog
```
postgres=# CREATE USER catalog;
```

3. Set a password for user catalog
``` 
postgres=# ALTER ROLE catalog WITH PASSWORD 'myNewPassword';
```
_I've also paste the myNewPassword on the "Notes to Reviewer" field._

4. Give user "catalog" permission to "catalog" application database
```
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
```

5. Quit postgreSQL 
```
postgres=# \q
```

Exit from user "postgres"
```
exit
```

#### Install git to clone Starship Catalog app project.

```shell
sudo apt-get install git
cd /var/www
sudo mkdir FlaskApp
cd FlaskApp/
git clone https://github.com/<YOUR USER HERE>/fsnd-p4-item-catalog.git
sudo git clone https://github.com/<YOUR USER HERE>/fsnd-p4-item-catalog.git
sudo mv ./fsnd-p4-item-catalog ./catalog
#to rename the project directory
cd catalog/
```

#### Configure the Catalog app

1. Rename ```application.py``` to```__init__.py```
``` shell
sudo mv application.py __init__.py
```
2. Change the engine for the database 

```python
# engine = create_engine('sqlite:///fleet.db?check_same_thread=False')
#to
create_engine('postgresql://catalog:myNewPassword@localhost/catalog')
```
at

```shell
sudo nano __init__.py 
sudo nano db_config.py 
```
3. Install pip

```shell
sudo apt-get install python-pip
sudo touch requirements.txt
sudo nano requirements.txt
#and paste the requirements on the box below
```
- Paste those dependencies at requirements.txt:
```
requests>=2.12
psycopg2
sqlalchemy
flask
Flask-Session
httplib2
oauth2client
flask_httpauth
```

```shell

sudo -H pip install -r requirements.txt
```
4. Install psycopg2 
```shell
pip install psycopg2==2.7.3.2
```

5. Create database schema 
```shell 
sudo python db_config.py
```

6. Configure and Enable a New Virtual Host

- Create FlaskApp.conf to edit: 
```shee
sudo nano /etc/apache2/sites-available/FlaskApp.conf
```

- Add the following lines of code to the file to configure the virtual host.
```
<VirtualHost *:80>
	ServerName ec2-18-216-71-137.compute-1.amazonaws.com
	ServerAdmin danielpaladar@gmail.com
	WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
	<Directory /var/www/FlaskApp/FlaskApp/>
		Order allow,deny
		Allow from all
	</Directory>
	Alias /static /var/www/FlaskApp/FlaskApp/static
	<Directory /var/www/FlaskApp/FlaskApp/static/>
		Order allow,deny
		Allow from all
	</Directory>
	ErrorLog ${APACHE_LOG_DIR}/error.log
	LogLevel warn
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Enable the virtual host with the following command: 
```shell
sudo a2ensite FlaskApp
```

7. Create the .wsgi File

```shell
cd /var/www/FlaskApp
sudo nano flaskapp.wsgi 
#to create the .wsgi File under /var/www/FlaskApp and edit it
``` 
- add the following code to flaskapp.wsgi
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from FlaskApp import app as application
application.secret_key = 'Add your secret key'
```
_Again, Add your secret key value will be into the "Notes to Reviewer" field._

- restart Apache
```
sudo service apache2 restart
```

## References and inspiration

https://www.postgresql.org/
http://httpd.apache.org/
https://wsgi.readthedocs.io/en/latest/
https://stackoverflow.com/questions/36394101/pip-install-locale-error-unsupported-locale-setting
https://github.com/dbcli/pgcli/issues/844
https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server

[jungleBadger/-nanodegree-linux-server](https://github.com/jungleBadger/-nanodegree-linux-server)

[SteveWooding/fullstack-nanodegree-linux-server-config](https://github.com/SteveWooding/fullstack-nanodegree-linux-server-config)

[bcko/Ud-FS-LinuxServerConfig-LightSail](https://github.com/bcko/Ud-FS-LinuxServerConfig-LightSail)
