# Udacity Configure Linux Server to Host Web Application  
This readme provides a step-by-step instruction to configure an Ubuntu 16.04.5 Linux virtual machine. I used a Lightsail instance, but this procedure should work more or less the same. 

## Information on Server  
IP Address: 34.232.193.51  
URL: [cory.linuxserver.34.232.193.51.xip.io](cory.linuxserver.34.232.193.51.xip.io)  
SSH Port: 2200  
  
# General Server Configuration and Security 
I used a [Lightsail](https://aws.amazon.com/lightsail/) instance to provide myself an Ubuntu Virtual Machine. 

## Add User Grader and Grant Sudo Privileges 
Add a user named **grader**:  
`sudo adduser grader`  
You will be prompted for a password, choose one you will remember.  It may come in handy later.  
  
Add user to sudo group:  
`usermod -aG sudo grader`  

[Reference](https://www.digitalocean.com/community/tutorials/how-to-create-a-sudo-user-on-ubuntu-quickstart)  
  
## Remove Password Request for user grader When Using Sudo  
Open the **/etc/sudoers** file:  
`sudo visudo`  

Add the following line to the bottom of the file:  
`grader ALL=(ALL) NOPASSWD: ALL`  
  
[Reference](https://askubuntu.com/questions/147241/execute-sudo-without-password)  

## Update System Packages  
Search for package updates:  
`sudo apt-get update`  
  
Apply updates:  
`sudo apt-get upgrade`  

## Allow user grader to Use SSH Key to Login  
Enter the following commands to enable SSH key login for user grader:
```
sudo mkdir /home/grader/.ssh 
sudo chmod 700 /home/grader/.ssh
sudo cp /root/.ssh/authorized_keys /home/grader/.ssh/
sudo chown grader:grader /home/grader/.ssh/authorized_keys
sudo chmod 644 /home/grader/.ssh/authorized_keys
```  
You can connect to the server as user grader with:  
`ssh grader@34.232.193.51 -p 22 -i ~/.ssh/udacity_key.pem`  
provided you have downloaded the provided SSH key to the .ssh folder. Alternatively, you can store it in your Downloads files and replace `.ssh` with `Downloads`.  

## Disable Root Login and Password Login  
Open the config file with the text editor while in root directory:  
`sudo nano /etc/ssh/sshd_config`  

Change the line from `PermitRootLogin prohibit-password` to `PermitRootLogin no`

Additionally, ensure the line `PasswordAuthentication no` is in the file. If not, change the yes to a no. The default for Lightsail instances is no. 

Enter the following to reset the ssh service and implement changes:  
`sudo service ssh restart` 

## Change SSH Port to 2200 from 22 
Edit config file:  
`sudo nano /etc/ssh/sshd_config`  
Change line from `Port 22` to `Port 2200`  
Restart SSH:  
`sudo service ssh restart` 

From now on, you must connect using Post 2200:  
`ssh grader@34.232.193.51 -p 2200 -i ~/.ssh/udacity_key.pem`

## Update Lightsail Firewall Settings and IP Address
1. Login to the Lightsail UI on AWS and navigate to the Networking tab.  
2. Click the **Create static IP** button and follow the on-screen steps to attach it to your Lightsail instance.  This is your new IP address to login to your instance and not worry about it changing on you because Lightsail instances are sometimes randomly assigned IP Addresses on reboot.
3. Navigate back to the instances tab and click on your instance. 
4. Click on Networking tab  
5. In the Firewall section, add 2 rules: Custom TCP 123, Custom TCP 2200. And delete SSH TCP 22.  

## Configure Uncomplicated Firewall  
Deny all incoming connections:  
`sudo ufw default deny incoming`  
  
Allow all outgoing connections:  
`sudo ufw default allow outgoing`  
  
Allow SSH, NTP, and HTTP:  
```
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
```

Before you turn on the firewall check that the intended rules were added:  
`sudo ufw show added`  
  
Provided the rules were added as intended, enable the firewall:  
`sudo ufw enable`  

Now, try logging into the server on a new terminal on your local computer using `ssh grader@34.232.193.51 -p 2200 -i ~/.ssh/udacity_key.pem` to make sure you did not block yourself out!  

[Allow NTP Reference](https://askubuntu.com/questions/709843/how-to-configure-ufw-to-allow-ntp-to-work)

## Update timezone to UTC  
Change timezone:  
`sudo timedatectl set-timezone UTC`

# Configure Server to Host Web Application  
## Install Required Packages and Applications  
Apache:  
`sudo apt-get install apache2`  
  
mod_wsgi:  
`sudo apt-get install libapache2-mod-wsgi`  

Python libraries:  
```
sudo apt-get install python-psycopg2 python-flask
sudo apt-get install python-sqlalchemy python-pip
sudo pip install oauth2client
sudo pip install requests
sudo pip install httplib2
```
  
Git:  
`sudo apt-get install git`

PostgresQL:  
`sudo apt-get install postgresql postgresql-contrib`  
  
Create a user **cars**:  
`sudo -u postgres createuser -P cars`  
Set the password to something you will remember because you will need it in the next steps.  
  
Create a database **cars**:  
`sudo -u postgres createdb -O cars cars`

[Reference for PostgresQL Setup](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-18-04)  
  
## Copy Item Catalog Project Repository from Github  
From the root, `/`, directory: 
```
sudo cd /var/www/html
sudo mkdir fsnd-server
sudo cd fsnd-server
sudo mkdir catalog
sudo -u www-data git clone https://github.com/crichard93/Item-Catalog.git catalog
sudo cd ..
sudo chown -R root:root fsnd-server
sudo chmod -R 755 fsnd-server
```
[Reference for owners and permissions](https://smarttechnicalworld.com/fix-forbidden-you-dont-have-permission-to-access-on-this-server/)  
  
## Create catalog.wsgi
From the **/var/html/www/fsnd-server/catalog** directory, enter:  
`sudo nano catalog.wsgi`  

Then save the following into the file:  
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/html/fsnd-server/")
from catalog import app as application
application.secret_key = '123youwi11neve4eve4hackthi5key'
```
[Reference for deploying a Flask Application](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)

## Preparing Item Catalog Application for Deployment  
From the **/var/html/www/fsnd-server/catalog** directory, edit **project_server.py** with `sudo nano project_server.py`.
Now, there are 3 lines to edit in this file. 
1. From:  
```
CLIENT_ID = json.loads(
    open('client_secrets.json', 'r').read())['web']['client$
APPLICATION_NAME = "Item Catalog Application"
```
to:  
```
CLIENT_ID = json.loads(
    open('/var/www/html/fsnd-server/catalog/client_secrets.json', 'r').read())['web']['client$
APPLICATION_NAME = "Item Catalog Application"
```
2. From:  
`engine = create_engine('sqlite:///cars.db?check_same_thread=False')`  
to  
`engine = create_engine('postgresql://cars:udacity@localhost/cars')`

3. From:  
```
 try:
        # Upgrade the authorization code into a credentials object
        oauth_flow = flow_from_clientsecrets('client_secret.json', scope='')
        oauth_flow.redirect_uri = 'postmessage'
```
to:  
```
 try:
        # Upgrade the authorization code into a credentials object
        oauth_flow = flow_from_clientsecrets('/var/www/html/fsnd-server/catalog/client_secrets.json', scope='')
        oauth_flow.redirect_uri = 'postmessage'
```
Finally, save the updated **project_server.py** file as **__init__.py** to initialize the application when the web page is visited.  

Next, update **populate_car_database.py** and **database_setup.py** with the same line from **2** above, minus the initial `?check_same_thread=False`.  

[Reference for accessing database URLs with SQLAlchemy](https://docs.sqlalchemy.org/en/rel_1_0/core/engines.html#postgresql)

## Populate the Database for the Server  
Create the database the server will use by running python script from directory **/var/html/www/fsnd-server/catalog**:  
`python database_setup.py`  

Populate the created database:  
`python populate_car_database.py`

## Update Google OAuth  
Go to [Google Developers Console](https://console.developers.google.com) and open the project associated with OAuth. Under OAuth consent screen, add**xip.io** to Authorized Domains.  
  
Then, under the credentials tab, edit the client ID. Add in the IP, **http://34.232.193.51**, and URL, 	**http://cory.linuxserver.34.232.193.51.xip.io** to the Authorized Javascript origins. Under Authorized redirect URIs, add the following URIs:  
1. http://cory.linuxserver.34.232.193.51.xip.io/login
2. http://cory.linuxserver.34.232.193.51.xip.io/brands
3. http://cory.linuxserver.34.232.193.51.xip.io/brands/

Press **Save**, then **DOWNLOAD JSON**. Open the downloaded file in a text editor.  

Back on the server terminal, open the file **/var/www/html/fsnd-server/catalog/client_secrets.json**.  
Delete all of the text currently in the file and replace it with the text from the **.json** file downloaded from the Google Developers Console.

## Apache2 Configuration to Serve Web Application  
Last, but not least, you need to set up a configuration file for Apache to serve the application.  
Create a new file `/etc/apache2/sites-available/catalog-app.conf` and add the following text to it:
```
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        # Utilize xip.io service to provide DNS for application
        Servername cory.linuxserver
        ServerAlias cory.linuxserver.*.xip.io

        # Define the location of the app's WSGI file
        WSGIScriptAlias / /var/www/html/fsnd-server/catalog/catalog.wsgi

        # Allow Apache to serve the WSGI app from the catalog app directory
        <Directory /var/www/html/fsnd-server/catalog/>
                Require all granted
        </Directory>

        # Allow Apache to serve the files from the static directory
        <Directory  /var/www/html/fsnd-server/catalog/static/>
                Require all granted
        </Directory>

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Disconnect the default file being served:
`sudo a2dissite 000-default.conf`
Then tell Apache2 to serve the new catalog application:
`sudo a2ensite catalog-app.conf`
Refresh Apache to implement the changes: 
`sudo service apache2 restart`

[Reference for setting up apache2](https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts)

[Reference for setting up xip.io DNS](https://stackoverflow.com/questions/12570759/how-to-use-xip-io-with-several-virtualhosts-and-server-names-local-dev)