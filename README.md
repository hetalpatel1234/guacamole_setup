# Guacamole Server Setup on Ubuntu
guacamole_setup_26_Dec_2024


## Step 1: Install Required Dependencies for Server 

Update the package list and install required dependencies:

```bash
sudo apt update
sudo apt install -y gcc nano vim curl wget g++ libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin libossp-uuid-dev
sudo apt install -y libavcodec-dev libavformat-dev libavutil-dev libswscale-dev build-essential libpango1.0-dev libssh2-1-dev libvncserver-dev libtelnet-dev libpulse-dev libvorbis-dev libwebp-dev
sudo apt update
sudo apt install -y freerdp2-dev freerdp2-x11 -y 
```


## Step 2: Apache Tomcat Installed 

we are going to install the Apache Tomcat Java servlet container which will run the Guacamole Java war file. Tomcat acts as the servlet container that runs Guacamole's web application, handling HTTP requests from the user’s browser. 

Java installed first: 
```bash
sudo apt install default-jdk 
``` 
Once it is installed, you can check the version installed 
```bash
java –version 
``` 
Install Apache Tomcat 
```bash
sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user 
sudo systemctl enable --now tomcat9 
sudo systemctl status tomcat9 
``` 
Tomcat listens on port 8080 by default and as you can guess, we need to allow access to the application remotely by allowing the port on the firewall. 
```bash
ufw allow 8080/tcp
``` 

## Step 3: Apache Tomcat Installed 

```bash
mkdir guacamole
cd guacamole
wget https://archive.apache.org/dist/guacamole/1.5.4/source/guacamole-server-1.5.4.tar.gz
tar xzf guacamole-server-1.5.4.tar.gz
cd guacamole-server-1.5.4
``` 
```bash
./configure --with-init-dir=/etc/init.d
``` 

![alt text](<Screenshot from 2024-12-26 13-17-42.png>)
```bash
make 
make install  
ldconfig 
mkdir -p /etc/guacamole/{extensions,lib} 
```
Create guacd.conf configuration file and edit
```bash
vim /etc/guacamole/guacd.conf
[daemon]
pid_file = /var/run/guacd.pid
#log_level = debug

[server]
#bind_host = localhost
bind_host = 127.0.0.1
bind_port = 4822

#[ssl]
#server_certificate = /etc/ssl/certs/guacd.crt
#server_key = /etc/ssl/private/guacd.key                                        
```

Refresh systemd for it to find the guacd (Guacamole proxy daemon) service installed in /etc/init.d/ directory.

```bash
systemctl daemon-reload
```
Once reloaded, start and enable the guacd service.

```bash
systemctl restart guacd
systemctl enable guacd
```
please check service status

```bash
systemctl status guacd
```


## Step 4: Install the Guacamole Web Application

There are two critical files involved in the deployment of Guacamole: guacamole.war, which is the file containing the web application, and guacamole.properties, the main configuration file for Guacamole.


```bash
cd guacamole
wget https://archive.apache.org/dist/guacamole/1.5.4/binary/guacamole-1.5.4.war
mv guacamole-1.5.4.war /var/lib/tomcat9/webapps/guacamole.war
```

Now you need to define how the Guacamole client will connect to the Guacamole server (guacd)
Create GUACAMOLE_HOME environment variable
```bash
vim /etc/default/tomcat
GUACAMOLE_HOME=/etc/guacamole
```
edit this file 
```bash
vim /etc/profile
export GUACAMOLE_HOME=/etc/guacamole
```
Create /etc/guacamole/guacamole.properties config file 
```bash
guacd-hostname: 127.0.0.1
guacd-port:     4822
```
## Step 5:  Setup Guacamole Database Authentication
two methodology we used 
1. user-mapping.xml
2. mysql database

put all details related user connection or remote destop device which we want to access
##
### 1. user-mapping.xml
create file user-mapping.xml
```bash
vim /etc/guacamole/user-mapping.xml
<user-mapping>
    <!-- Allow access to a specific SSH server -->
    <authorize username="guacadmin" password="guacadmin">
        <connection name="Linux-Server">
            <protocol>ssh</protocol>
            <param name="hostname">192.168.x.x</param>  <!-- Only this remote system -->
            <param name="port">22</param>
            <param name="username">username</param>
            <param name="password">passwd</param>
        </connection>
    </authorize>
</user-mapping>
``` 

```bash
vim /etc/guacamole/user-mapping.xml
<?xml version="1.0" encoding="UTF-8"?>

<user-mapping>
    <authorize username="testuser" password="testpassword">
        <connection name="My VNC Connection">
            <protocol>vnc</protocol>
            <param name="hostname">192.168.x.x</param>
            <param name="port">5900</param>
            <param name="password">vnc server passwd </param>
        </connection>
    </authorize>
</user-mapping>
``` 
open file and append data in file guacamole.proprties
 ```bash
 vim /etc/guacamole/guacamole.properties
 user-mapping:  /etc/guacamole/user-mapping.xml
 ``` 
 ```bash
sudo systemctl restart tomcat9 guacd
 ``` 

### 2. mysql database 

 ```bash
sudo apt update
sudo apt upgrade
apt install mysql-server
systemctl start mysql.service
 ```

 Now log in to the MySQL console
 ```bash
mysql -uroot -p
 ```
after access a mysql concol Create database
create a user and create a password for that  
```bash
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
```
```bash
GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD on *.* TO 'username'@'localhost' WITH GRANT OPTION;
```
```bash
FLUSH PRIVILEGES;
exit;
```
Restart MySQL Service
```bash
sudo systemctl restart mysql
```

now go to the guacamole directory and MySQL Connector/J (Java Connector)
```bash
cd guacamole
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.3.0.tar.gz
tar -xf mysql-connector-j-8.3.0.tar.gz
cp mysql-connector-j-8.3.0/mysql-connector-j-8.3.0.jar /etc/guacamole/lib/
wget https://downloads.apache.org/guacamole/1.5.4/binary/guacamole-auth-jdbc-1.5.4.tar.gz
tar -xf guacamole-auth-jdbc-1.5.4.tar.gz
mv guacamole-auth-jdbc-1.5.4/mysql/guacamole-auth-jdbc-mysql-1.5.4.jar /etc/guacamole/extensions/
```

Switch to the extracted JDBC plugin path:

```bash
cd guacamole-auth-jdbc-1.5.4/mysql/schema
```
Import the SQL schemas:  add mysql database name  insted of mysql_database_name 
```bash
cat *.sql | mysql -u root -p (mysql_database_name)
```
modify the properties of Guacamole with mysql data 
open a file and append data 
```bash
vim /etc/guacamole/guacamole.properties
mysql-hostname: localhost 
mysql-port: 3306
mysql-database: mysql_database_name 
mysql-username: mysql_database_user 
mysql-password: mysql_database_passwd
```
```bash
systemctl restart tomcat9 guacd mysql
```


## Step 6: Accessing Guacamole Web Interface

set ip-or-domain = your system ip 
```bash
 http://ip-or-domain-name:8080/guacamole
```

create a connection 
![alt text](<Screenshot from 2024-12-26 15-00-14.png>)
