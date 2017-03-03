---
layout: post
title: How To Flask, Python, Centos7, Apache, uWSGI
---

# How To: Flask, Python, CentOS7, Apache and uWSGI
Following is a walk-through for deploying a simple python-flask application with Linux Centos7, apache, virtualenv and uWSGI



## This shouldn't have been so hard to pull together
But it was, so i'm documenting for your reading pleasure

I've been questing for python literacy lately.  I've been developing a flask website, but had problems deploying it under centos7.  I come from the world of perl, where it's all roses and sunshine, so I'm always suprised when deployments fail.  This failed in spectacularly weird ways.

Before I attack those problems, I want to see a simple flask app deploy under these conditions:
* Fresh VPS instance under CentOS7
* Python3 from official repo (EPEL repo carries python 3.4)
* Apache as the web server.  (I have much love for apache, shut up about nginx)
* Application operating under a virtualenv
* Application operating under an unprivileged user account other than apache
* Installed configuration stable across reboots and package upgrades
* Configuration compatible with installation into another VPS I maintain for pet projects, so it doesn't require it's own server

Once I made this happen, I have a template for hacking deployment problems with my other sprawling flask pet projects.  Online docs for flask and wsgi sent me down several dead-end rabbit-holes. But it always pulls together in the end.

If you follow these directions, you will have a working sample flask app.
I'm going to deploy this sample as a linode stackscript for one-click deployments of this configuration




## Basic Server Preparation
This isn't necessary, but you should never spin up a linux instance and not secure it.
```
# Appropriate values should be plugged in here
export WHITELIST_IPADDR=0.0.0.0
export MARIADB_ROOT_PASSWORD=listen_to_sneakerpimps


# Lock down the firewall immediately
# - Block all incoming connections, even ssh
# - Allow all incoming from our specified IP
#   SSH is only going to function from this allowed IP
# - Allow incoming connections on web ports
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --zone=public --add-service=http
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=trusted --add-source=${WHITELIST_IPADDR}
firewall-cmd --zone=trusted --add-source=${WHITELIST_IPADDR} --permanent
firewall-cmd --zone=public --remove-service=ssh
firewall-cmd --zone=public --remove-service=ssh --permanent


# Update system, install LAMP packages
yum -y update
yum -y install epel-release
yum -y install \
    exim \
    httpd \
    mailx \
    mariadb \
    mariadb-server \
    nano \
    psacct
yum clean all


# SSH Hardening
sed -i 's/^#?PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#?AllowTcpForwarding yes/AllowTcpForwarding no/' /etc/ssh/sshd_config
sed -i 's/^#?ClientActiveCountMax 3/ClientActiveCountMax 2/' /etc/ssh/sshd_config
sed -i 's/^#?Compression DELAYED/Compression no/' /etc/ssh/sshd_config
sed -i 's/^#?LogLevel INFO/LogLevel VERBOSE/' /etc/ssh/sshd_config
sed -i 's/^#?MaxAuthTries 6/MaxAuthTries 1/' /etc/ssh/sshd_config
sed -i 's/^#?MaxSessions 10/MaxSessions 2/' /etc/ssh/sshd_config
sed -i 's/^#?TCPKeepAlive yes/TCPKeepAlive no/' /etc/ssh/sshd_config
sed -i 's/^#?UseDNS yes/UseDNS no/' /etc/ssh/sshd_config
sed -i 's/^#?X11Forwarding yes/X11Forwarding no/' /etc/ssh/sshd_config
sed -i 's/^#?AllowAgentForwarding yes/AllowAgentForwarding no/' /etc/ssh/sshd_config


# Deploy a login banner
cat << EOF >> /etc/issue
###############################################################
#                [waste of space]                             #
#      All connections are monitored and recorded             #
# Disconnect IMMEDIATELY if you are not an authorized user!   #
###############################################################
EOF
cp /etc/issue /etc/issue.net

systemctl reload sshd


# Configure exim for outgoing mail delivery for local apps
# (your SPF records should be set)
sed -i 's/= @ :/= /' /etc/exim/exim.conf
alternatives --set mta /usr/sbin/sendmail.exim
systemctl enable exim
systemctl start exim



# Configure mariadb
systemctl enable mariadb
systemctl start mariadb
mysql -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');"
mysql -e "DELETE FROM mysql.user WHERE User='';"
mysql -e "DROP DATABASE test;"
mysql -e "UPDATE mysql.user SET Password=PASSWORD('$MARIADB_ROOT_PASSWORD') WHERE User='root';"
mysql -e "FLUSH PRIVILEGES;"


```




## Install Python and prereqs
```
yum -y install python34 python34-pip
# DO NOT use yum -y install python-virtualenv

sudo yum -y install python34 python34-devel python34-pip mod_proxy_uwsgi gcc
sudo pip3 install --upgrade pip
sudo pip3 install virtualenv flask uwsgi
```




## Flask Demo Application
Let's create the least exciting flask demo application possible.

### Create user and virtualenv for app.
```
# Create the unprivileged user
sudo adduser flaskdemo
sudo passwd flaskdemo
sudo su flaskdemo

# Create our project directory
cd
mkdir flaskdemo
cd flaskdemo

# init and activate a python-virtualenv with prereqs
virtualenv -p python3 env
source env/bin/activate

```

### Create File: ~/flaskdemo/flaskdemo.py
This is our demo application.  Exciting.
```
#!/usr/bin/env python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def root():
    return "You have found somewhere on your road to nowhere.<br>Congrats"

if __name__ == "__main__":
    app.run(host='0.0.0.0')


```

### Create File: ~/flaskdemo/wsgi.py
This is the wsgi wrapper.  Not strictly useful here, but will be very useful for a real flask app
```
from flaskdemo import app as application

if __name__ == "__main__":
    application.run()

```

### Create File: ~/flaskdemo/flaskdemo.ini
Configuration options for the uwsgi process.  
mod_proxy_uwsgi doesn't support sockets until apache 2.4.9.  Currently, CentOS is sporting apache 2.4.6 
Sockets would be faster, but we're using network transport instead on port 8000
```
[uwsgi]
module = wsgi

master = true
processes = 5

socket = 127.0.0.1:8000
chmod-socket = 660
vacuum = true

die-on-term = true

```




## Test sample flask app
If you want, test yourself for typos. 
`uwsgi --socket 0.0.0.0:5000 --protocol=http -w wsgi`



## Create system service for uWSGI
### /etc/systemd/system/flaskdemo.service
```
[Unit]
Description=uWSGI server for flaskdemo
After=network.target

[Service]
User=flaskdemo
Group=apache
WorkingDirectory=/home/flaskdemo/flaskdemo
Environment="PATH=/home/flaskdemo/flaskdemo/env/bin"
ExecStart=/home/flaskdemo/flaskdemo/env/bin/uwsgi --ini flaskdemo.ini

[Install]
WantedBy=multi-user.target
```

### Start the system service
```
sudo systemctl enable flaskdemo
sudo systemctl start flaskdemo
```



## Configure and launch Apache httpd
### append to end of /etc/httpd/conf/httpd.conf
```
LoadModule proxy_uwsgi_module modules/mod_proxy_uwsgi.so
<VirtualHost *>
    ServerName flaskdemo.com
    ProxyPass / uwsgi://127.0.0.1:8000
</VirtualHost>

```
### Test the apache config
`httpd -t`
If this doesn't return OK, I've steered you wrong, pal

### Start apache
```
sudo systemctl enable httpd
sudo systemctl start httpd
```




## We have a functioning, flask app under apache and centos7!
Test your program and rejoice


## References
* [flask-uwsgi-nginx-centos7](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-centos-7)
* [apache docs-uwsgi](http://uwsgi-docs.readthedocs.io/en/latest/Apache.html)
* [flask docs-uwsgi](http://flask.pocoo.org/docs/0.12/deploying/uwsgi/)
* [uwsgi-docs](http://uwsgi-docs.readthedocs.io/en/latest/Configuration.html)
