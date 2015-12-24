# Configuring-Linux-Web-Server
Full-Stack Web Nanodegree ( Project 5 )

The project is to configure linux web server and deploy [project 3](https://github.com/AungThiha/catalog).<br>
The project consists of:<br>
* Update all currently installed packages
* Create new user and add it to sudoers
* Disable remote login of the root user
* Enfore key-based ssh authentication
* Change the SSH port from 22 to 2200
* Configure the local timezone to UTC
* Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
* Configure the firewall to monitor for repeat unsuccessful login attempts and appropriately ban attackers
* Include cron script to automatically manage package updates.
* Install and configure Apache to serve a Python mod_wsgi application
* Install and configure PostgreSQL:
  * Do not allow remote connections
  * Create a new user named catalog that has limited permissions to your catalog application database
* Install git, clone and setup your Catalog App project (from your GitHub repository from earlier in the Nanodegree program) so that it functions correctly when visiting your server’s IP address in a browser. Remember to set this up appropriately so that your .git directory is not publicly accessible via a browser!
* Includes monitoring applications that provide automated feedback on application availability status and/or system security alerts.


## IP Address
52.34.3.178


## SSH port
2200


## URL for hosted web application
http://52.34.3.178/


## Summary of software installed and configuration changes made

### Create new user and add it to sudoers
* create a user named **grader** and add it to sudoers by following the tutorial offered at Udacity.
* when issueing **sudo** with this account, it will ask for password. The password is provided in "Notes to Reviewer" field during the submission process.

### Disable remote login of the root user
1. move the file named **authorized_keys** from **/root/.ssh/** to **/home/grader/.ssh/**
1. change ownership and group of **authorized_keys**
  * $ sudo chown grader:grader /home/grader/.ssh/authorized_keys
1. modify **/etc/ssh/sshd_config**  to disable root login. The modification is at line 28.<br>
  ```
  PermitRootLogin no
  ```

### Change the SSH port from 22 to 2200
1. modify **/etc/ssh/sshd_config**. The modification is at line 5.<br>
  ```
  Port 2200
  ```

### Solve 'apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1.'
solve it by typing following commands:
* $ echo "ServerName localhost" | sudo tee /etc/apache2/conf-available/fqdn.conf
* $ sudo a2enconf fqdn

### Monitor for repeat unsucessful login attempts and ban attackers
1. Install fail2ban
  * $ sudo apt-get install fail2ban
1. type the following command.
  * $ sudo cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/jail.local 
1. Major changes of **/etc/fail2ban/jail.local** are at line 120.
  ```
  [ssh]
  
  enabled  = true
  port     = 2200
  filter   = sshd
  logpath  = /var/log/auth.log
  bantime  = 3600
  findtime = 600
  maxretry = 3
  ```
1. then, type the following commands.
  * $ sudo iptables -A INPUT -i lo -j ACCEPT
  * $ sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  * $ sudo iptables -A INPUT -p tcp --dport 2200 -j ACCEPT
  * $ sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
  * $ sudo iptables -A INPUT -j DROP
  * $ sudo service fail2ban stop
  * $ sudo service fail2ban start
1. Now, all is set to monitor for repeat unsuccessful login attempts and appropriately ban attackers.

reference:<br>
* [How To Protect SSH with Fail2Ban on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)

### Include cron script to automatically manage package updates
Updating all the packages may break the system. So, the cron job to update only security packages is added instead.

1. install cron-apt and anacron
  * $ sudo apt-get install cron-apt
  * $ sudo apt-get install ancron
1. create **/etc/cron.weekly/apt-security-updates**
  * $ sudo touch /etc/cron.weekly/apt-security-updates
1. **/etc/cron.weekly/apt-security-updates** is needed to apply update security packages.
  * copy the following text to it.
  ```
  echo "**************" >> /var/log/apt-security-updates
  date >> /var/log/apt-security-updates
  aptitude update >> /var/log/apt-security-updates
  aptitude safe-upgrade -o Aptitude::Delete-Unused=false --assume-yes --target-release `lsb_release -cs`-security >> /var/log/apt-security-updates
  echo "Security updates (if any) installed"
  ```
1. define permission for **/etc/cron.weekly/apt-security-updates**.
  * $ sudo chmod +x /etc/cron.weekly/apt-security-updates
1. use the logrotate utility to prevent the log file, **/var/log/apt-security-updates**, from getting too large.
  * crate **/etc/logrotate.d/apt-security-updates** and copy the following text to it.
  ```
  /var/log/apt-security-updates {
    rotate 2
    weekly
    size 250k
    compress
    notifempty
  }
  ```

reference:<br>
[AutomaticSecurityUpdates](https://help.ubuntu.com/community/AutomaticSecurityUpdates)

### Install and configure Apache to serve a Python mod_wsgi application
1. install libapache2-mod-wsgi
  * $ sudo apt-get install libapache2-mod-wsgi
1. modify **/etc/apache2/sites-enabled/000-default.conf**.
  * the modification is at line 13.
  ```
        WSGIScriptAlias / /var/www/FlaskApp/myapp.wsgi
        Alias /uploads /var/www/FlaskApp/catalog/uploads
        <Directory /var/www/FlaskApp/catalog>
                Order deny,allow
                Allow from all
        </Directory>
  ```
1. create FlaskApp directory.
  * $ sudo mkdir /var/www/FlaskApp
1. change working directory to **/var/www/FlaskApp**
  * $ cd /var/www/FlaskApp
1. create **myapp.wsgi**.
  * $ sudo touch myapp.wsgi
1. paste the following to **myapp.wsgi**.
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from catalog.application import app as application
application.secret_key = 'MySuperSecretKez3llap3kzniyoq'
```

### Install and configure PostgreSQL
1. install postgresql
  * $ sudo apt-get install postgresql
1. connect to postgresql as postgres
  * $ sudo su - postgres
1. create a role named **catalog** with password. The password is provided in "Notes to Reviewer" field during the submission process.
1. create a database named **catalog**.
1. grant database **catalog** to role **catalog**.
1. remote connection is disabled by default. it can be confirmed by typing the follwing command.
  * $ sudo cat /etc/postgresql/9.3/main/pg_hba.conf
1. then, type the following commands.
  * $ sudo apache2ctl restart
  * $ sudo service apache2 reload
  * $ sudo service apache2 restart

reference:
[How To Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

### Install git, clone and setup your Catalog App project
the app is located at **/var/www/FlaskApp/catalog**

### Hide .git folder from public
1. change working directory to **/var/www/FlaskApp/catalog/.git/**
  * $ cd /var/www/FlaskApp/catalog/.git/
1. create **.htaccess** file.
  * $ sudo touch .htaccess
1. copy and paste the following text to **/var/www/FlaskApp/catalog/.git/.htaccess**.
```
Order allow,deny
Deny from all
```

### Includes monitoring applications
1. install glances
  * $ sudo apt-get install glances


## A list of any third-party resources used
* [How To Protect SSH with Fail2Ban on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
* [AutomaticSecurityUpdates](https://help.ubuntu.com/community/AutomaticSecurityUpdates)
* [How To Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
* [Install ‘Glances’ (system monitor) on Ubuntu 13.04](http://www.hecticgeek.com/2013/06/install-glances-ubuntu-13-04/)