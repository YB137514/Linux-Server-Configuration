# Linux Server Configuration
How to deploy Flask web application to a virtual server in Amazon’s Elastic Compute Cloud (EC2).
Step by step insructions for Ubuntu 14.04.3 LTS(Trusty Tahr)

## Sample application:

URL for application | [link][12]
----------------------|-----------------------------
IP address of the server | 52.32.222.26
SSH Port | 2200

## Perform Basic Linux server configuration
1. Launch Virtual Machine and log in through the terminal
    *  Initial log in is done through root on default port

2. Create a new user named grader and grant this user sudo permissions.

    1| `adduser grader` |Add a new user grader
    ---|---------------- |-----------------------------------------------------------------
    2|`apt-get install finger` | Nice program to check that the user is added sucessfully
    3|`finger grader` | Checks that the new user was created 

3. Update all currently installed packages.

    `sudo apt-get update` | Update packaging source list
    ----------------------|-----------------------------
    `sudo apt-get upgrade` | To actually upgrade the packages 


4. Configure the local timezone to UTC.

    By default Ubuntu timezone is configured to UTC
    `sudo dpkg-reconfigure --frontend noninteractive tzdata`

## Secure Linux Server 

5. Change the SSH port from 22 to 2200
    Source: [KnownHost Wiki][2]

    1|`sudo nano /etc/ssh/sshd_config` | Open the configuration file
    ---|--------------------------------|---------------------------
    2|`Port 2200` |Change from Port 22 | 
    3|`CTRL+O Enter CTRL+X` | Save and close
    4|`sudo service ssh restart` | Restart the service for the changes to take effect
    5|`ssh -i ~/.ssh/catalog grader@52.32.222.26 -p 2200` |If all is well need to make sure that we are connecting on port 2200 and not on default port 22
    6|`sudo ufw deny 22` | Configure UFW to deny all connections on port 22 
    Source: [Digital Ocean][1]



6. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections:

    1 |`sudo ufw status`| To check the status of UFW
    ---|-----------------|---------------------------
    2 |`sudo ufw default deny incoming`|  Block all incoming request
    3 |`sudo ufw default allow outgoing` | Allow all outgoing
    4 |`sudo ufw status` | Warning: If we turn on the firewall now the server will be dead
    
    
     * for SSH (port 2200)
    
    5 |`sudo ufw allow 2200/tcp` |To change the port from default 22 to 2200
    ---|-------------------------|----------------------------------------
    
      * for HTTP (port 80)
    
    6 |`sudo ufw allow www` | To allow HTTP request
    ---|-------------------------|----------------------------------------
    
      * for NTP (port 123)
    
    For time syncronization with NTP on Ubuntu see: [Ubuntu][10]
    
    7|`sudo apt-get install ntp`| To install NTP. Source [Ubuntu][9]
    ---|-------------------------|----------------------------------------
    8|`sudo /etc/init.d/ntp status`| check if the server is running Source[Ubuntu][8]
    9| `sudo ufw allow ntp`| Allow NTP, uses default port 123
    10 |`sudo ufw enable`| To turn firewall on
    11 |`sudo ufw status` |To see all the rules that are engaged

      Source: [Digital Ocean][3]

## Install application

7. Install and configure Apache to serve a Python mod_wsgi application

      `sudo apt-get install apache2`| Install Apache server
   |--------------------------------|--------------------
      `sudo apt-get install libapache2-mod-wsgi` |Install application handler - mod_wsgi

7. Setup Flask directory to serve application

      1|`cd /var/www`|Change the directory to www where the main application folder will be located
      ---|-----------------|---------------------------
      2|`sudo mkdir Catalog`|Make a directory name Catalog
      3|`cd Catalog`|Move into Catalog directory
      4|`sudo apt-get install git`|Install git
      
      If copying application from remote repository such as git:
   
      5|`sudo git clone https://github.com/XXXXX/XXXXXX.git`| Clone the repository to folder
   ----------|-----------------|---------------------------
      
      If copying application from local computer use `scp` command to copy 
      
      6|`cd Catalog`|Move inside Catalog directory
   --------|-----------------|---------------------------
      7|`sudo apt-get install python-pip`|Install PIP
      8|`sudo pip install virtualenv` |Install virtual env
      9|`sudo virtualenv virtenv1`|Give the name of your virtual env, let's call it virtenv1
      10|`source virtenv1/bin/activate` |Activate the virtual environment with the following command
      11 |`sudo pip install Flask`|Install Flask inside the virtual environment
      12| `sudo pip install -r requirements.txt`|Install all of the requirements for the application 
      13|`sudo python application.py`|Run the application to see that it is working
      14|`sudo nano /etc/apache2/sites-available/Catalog.conf`|Configure new virtual host by copying the following to the Catalog.conf file
      
      ```
      <VirtualHost *:80>
                      ServerName mywebsite.com
                      # Use the public address of virtual server
                      ServerAlias ec2-52-32-222-26.us-west-2.compute.amazonaws.com
                      ServerAdmin admin@mywebsite.com
                      # Configure the root of the application
                      WSGIScriptAlias / /var/www/Catalog/catalog.wsgi
                      # Configure Catalog directory
                      <Directory /var/www/Catalog/Catalog/>
                              Order allow,deny
                              Allow from all
                      </Directory>
                      # Configure static directory
                      Alias /static /var/www/Catalog/Catalog/static
                      <Directory /var/www/Catalog/Catalog/static/>
                              Order allow,deny
                              Allow from all
                      </Directory>
                      ErrorLog ${APACHE_LOG_DIR}/error.log
                      LogLevel warn
                      CustomLog ${APACHE_LOG_DIR}/access.log combined
      </VirtualHost>
      
      ```
      15|`sudo a2ensite Catalog`|Enable the virtual host with this command. The opposite of this command, i.e. disables the site is sudo a2dissite Catalog
      ----------|-----------------|---------------------------
      16|`service apache2 reload`| Reload apache. If this command does not work, try restarting
      17|`sudo nano catalog.wsgi` |Create this file and copy the following:
      
      ```
      #!/usr/bin/python
      import sys
      import logging
      logging.basicConfig(stream=sys.stderr)
      sys.path.insert(0,"/var/www/Catalog/")
      
      from Catalog import app as application
      application.secret_key = 'Insert a secret Key for your application'
      
      ```
      18|`service apache2 restart`| If something does not work try restarting the server
      ----------|-----------------|---------------------------
      
      Source: [Digital Ocean][5]

8. Install and configure PostgreSQL:

      1|`sudo apt-get install postgresql`|Install PostgreSQL database
      ---|-------------------------|----------------------------------------

  1. Do not allow remote connections

            By default remote connections are not allowed. See the configuration file:
            `cat /etc/postgresql/9.3/main/pg_hba.conf`

  2. Create a new user named catalog that has limited permissions to your catalog application database

      1|`createuser -DRS catalog`|-D means that the new user will not be allowed to create databases, -R means that the new user will not be allowed to create new roles, -S means that  new user will not be a superuser. This is the default.
   ----------|-------------------------|-----------------------------------------
      2|`createdb -O grader catalogwithusers`|Create a database `catalogwithusers` which is owned by `grader`: This is created on the command line
      3|`sudo -u postgres bash`|Get a shell for the database superuser 'postgres'.
      4|`psql catalogwithusers`|login to database
      5|`GRANT ALL PRIVILEGES ON DATABASE catalogwithusers TO catalog;`| Grant limited privileges to user
      Optional|`GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO catalog;`|
      Optional|`GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO catalog;`|Source:[stackexchange][6]
      6|`ALTER USER "catalog" WITH PASSWORD 'Insert password here';`| Give password.  Source:[stackoverflow][7]
      7|`sudo nano /etc/postgresql/9.3/main/pg_hba.conf`| Replace `peer` to `md5` for local not, admin to allow connections with passwords 
      8|`sudo service postgresql restart`| Restart database
      9| Copy project files to a virtual server
      
         1. Code is stored in a remote repository like GitHub
            * Install git, clone and set up project if the code is stored in a remote repository like GitHub
               
               1|`sudo apt-get install git`| Install github
         -------|-----------------------|----------------------------------------
               2|`git clone https://github.com/User_Name/Repository_name`| Clone existing repository


         2. Code is stored on a local machine
            * Use scp command from terminal to securely copy files over ssh

10. Configure third party authentication.
      * Instructions for third party authenticatin are found on the [Readme][11] page of the catalog project.
      * Amazon EC2 Instance's public URL will look something like this: http://ec2-XX-XX-XXX-XXX.us-west-2.compute.amazonaws.com/ where the X's are replaced with your instance's IP address. 

## Additional Functionality
1. Firewall is configured to monitor for unsuccessful login attempts

      1|`sudo apt-get update`| Update existing packages
      ---|-------------------------|----------------------------------------
      2|`sudo apt-get install fail2ban`| Downloan `fail2ban'
      3|`sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`| create a copy of default config file
      4|`sudo nano /etc/fail2ban/jail.local`| make the following edits:
       For ssh:
       
        `port = 2200`| Replace from default ssh port to 2200
        ---------------------|----------------------------------------
       For apache:
       
        `enabled  = true`| enable log in protection for apache
        --------------------|----------------------------------------
      
      5|`sudo service fail2ban stop`| Stop the service
      ---|-------------------------|----------------------------------------
      6|`sudo service fail2ban start`| Restart the service
      
      Source: [Digital Ocean][15]

2. Automatic Updates

      1|`sudo apt-get install unattended-upgrades`| package to automatically install udates
      ---|-------------------------|----------------------------------------
      2|`sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`| default is that only security updates are downloaded and installed. 
      3|`sudo nano /etc/apt/apt.conf.d/10periodic`| configure the frequency with which updates are downloaded and installed 
      
      Source: [Ubuntu][14]

3. System Monitoring Tool -- Glances

      1|sudo apt-get install python-pip build-essential python-dev| Install requirements
      ---|-------------------------|----------------------------------------
      2| sudo pip install Glances| install application 
      
      Sources: [askubuntu][16], [Glances documentation][17]

## A summary of software installed

See `system_wide.txt` and `app_wide.txt` requirements files



[1]: https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server
[2]: https://wiki.knownhost.com/security/misc/how-can-i-change-my-ssh-port
[3]: https://www.digitalocean.com/community/tutorials/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server
[4]: http://ec2-52-32-222-26.us-west-2.compute.amazonaws.com/
[5]: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
[6]: http://dba.stackexchange.com/questions/33943/granting-access-to-all-tables-for-a-user
[7]: http://stackoverflow.com/questions/17443379/psql-fatal-peer-authentication-failed-for-user-dev
[8]: https://help.ubuntu.com/community/UbuntuTime
[9]:https://help.ubuntu.com/lts/serverguide/NTP.html
[10]: https://help.ubuntu.com/lts/serverguide/NTP.html#ntp-status
[11]: https://github.com/YB137514/Catalog-App/#quick-start
[12]: http://107.178.223.130/
[13]: https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
[14]:https://help.ubuntu.com/lts/serverguide/automatic-updates.html#update-notifications
[15]:https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04
[16]:http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu
[17]:http://glances.readthedocs.org/en/latest/index.html
