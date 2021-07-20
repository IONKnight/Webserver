#!/bin/bash

## Update System
yum check-update
yum update -y

## Add User
adduser flynn
echo "flynn" | passwd flynn --stdin
su flynn
cd /home/flynn
mkdir .ssh
chmod 700 .ssh
touch /home/flynn/.ssh/authorized_keys
cp /home/centos/.ssh/authorized_keys /home/flynn/.ssh/authorized_keys
chmod 600 .ssh/authorized_keys
## Customise Bashrc
export PS1="\[\e[32m\][\[\e[m\]\[\e[31m\]\u\[\e[m\]\[\e[33m\]@\[\e[m\]\[\e[32m\]\h\[\e[m\]:\[\e[36m\]\w\[\e[m\]\[\e[32m\]]\[\e[m\]\[\e[32;47m\]\\$\[\e[m\] "

## Change message of the day
echo "WARNING: Testing Environment" > /etc/motd

flynn ALL=(ALL) NOPASSWD:/sbin/reboot

## Create memory hogs.sh file
echo "ps -eo pid,%mem --sort=%mem | tail -n 10" > /home/flynn/memory_hogs.sh
chmod 700 memory_hogs.sh

## JDK 11
wget https://corretto.aws/downloads/resources/11.0.9.12.1/amazon-corretto-11.0.9.12.1-linux-x64.tar.gz
tar -C /opt/jdk -zxvf amazon-corretto-11.0.9.12.1-linux-x64.tar.gz
sudo update-alternatives --install /usr/bin/java java /opt/jdk/amazon-corretto-11.0.9.12.1-linux-x64 1

## Download Tomcat
cd /home/flynn
su flynn
wget https://ftp.heanet.ie/mirrors/www.apache.org/dist/tomcat/tomcat-9/v9.0.50/bin/apache-tomcat-9.0.50.zip
unzip apache-tomcat-9.0.50.zip
# Copy tomcat.service to /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat

# install nginx
# copy nginx.repo to /etc/yum.repos.d/
sudo yum update
sudo yum install nginx
sudo nginx
sudo systemctl start nginx
sudo systemctl enable nginx
# Create SSL
sudo mkdir /etc/ssl/private
sudo chmod 700 /etc/ssl/private
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
# Country Name (2 letter code) [AU]:UK
# State or Province Name (full name) [Some-State]:London
# Locality Name (eg, city) []:London
# Organization Name (eg, company) [Internet Widgits Pty Ltd]:Taskize
# Organizational Unit Name (eg, section) []:Taskize
# Common Name (e.g. server FQDN or YOUR name) []:server_IP_address
# Email Address []:admin@your_domain.com
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
# Modify homepage
sudo nano /usr/share/nginx/html/index.html
# redirect and point to tomcat
# Password protect nginx
sudo sh -c "echo -n 'flynn:' >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"
# copy tomcat.conf to /etc/nginx/conf.d and delete default.conf
nginx -t
nginx -s reload
# Restrict 8080 apart from localhost
iptables -A INPUT -p tcp -s localhost --dport 8080 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -j DROP


## Postgresql

sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
#
cd /var/lib/pgsql/9.6/data/
sudo nano postgresql.conf
# uncomment listen_addresses = 'localhost'
# uncomment port = 5432
su postgres
# psql into database
create database taskize;
create user flynn with encrypted password 'flynn';
grant all privileges on database taskize to flynn;
CREATE TABLE accounts (	user_id serial PRIMARY KEY,	username VARCHAR ( 50 ) UNIQUE NOT NULL,password VARCHAR ( 50 ) NOT NULL,email VARCHAR ( 255 ) UNIQUE NOT NULL);

INSERT INTO accounts (username, password, email) VALUES('dave', 'password', 'dave@google.com');

INSERT INTO accounts (username, password, email) VALUES('nick', 'password', 'nickgoogle.com');

INSERT INTO accounts (username, password, email) VALUES('mason', 'password', 'mason@google.com');
# create bash script display-tables.sh insert following line
echo 'psql -U postgres -d taskize -c "SELECT * from accounts"' > display-tables.sh
chmod 700 display-tables.sh
