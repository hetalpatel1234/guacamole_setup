# Access an android mobile using guacamole server 



## Step 1: Install Required Dependencies for Server 

Update the package list and install required dependencies:

```bash
apt update && apt upgrade -y
apt install -y gcc nano vim curl wget g++ libcairo2-dev libjpeg-turbo8-dev libpng-dev libtool-bin libossp-uuid-dev
apt install -y libavcodec-dev libavformat-dev libavutil-dev libswscale-dev build-essential libpango1.0-dev libssh2-1-dev libvncserver-dev libtelnet-dev libpulse-dev libvorbis-dev libwebp-dev
apt update
apt install -y freerdp2-dev freerdp2-x11 -y 
```


## Step 2: Apache Tomcat Installed 

we are going to install the Apache Tomcat Java servlet container which will run the Guacamole Java war file. Tomcat acts as the servlet container that runs Guacamole's web application, handling HTTP requests from the user’s browser. 

Java installed first: 
```bash
sudo apt install default-jdk 
``` 
Once it is installed, you can check the version installed 
```bash
java --version 
``` 
Install Apache Tomcat 
```bash
sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user 
sudo systemctl enable --now tomcat9 
sudo systemctl status tomcat9 
```
![alt text](<tomcat_status.png>)

Tomcat listens on port 8080 by default and as you can guess, we need to allow access to the application remotely by allowing the port on the firewall. 
```bash
ufw allow 8080/tcp
``` 

## Step 3: guacd gucamole server  install 

### Download and Compile guacd:
```bash
mkdir guacamole
cd guacamole
wget https://archive.apache.org/dist/guacamole/1.5.4/source/guacamole-server-1.5.4.tar.gz
tar xzf guacamole-server-1.5.4.tar.gz
cd guacamole-server-1.5.4
./configure --with-init-dir=/etc/init.d
```
![alt text](<configure.png>)
```bash
make
sudo make install
sudo ldconfig
```

### Configure guacd:

Create directories:
```bash
sudo mkdir -p /etc/guacamole/{extensions,lib}
```

Edit the configuration file:
```bash
sudo vim /etc/guacamole/guacd.conf
```

Add the following content:
```
[daemon]
pid_file = /var/run/guacd.pid

![alt text](<server_config.png>)


[server]
bind_host = 127.0.0.1
bind_port = 4822
```


Reload systemd and enable guacd:
```bash
systemctl daemon-reload
systemctl restart guacd
systemctl status guacd
```

---

## Step 4: Install the Guacamole Web Application

### Deploy the Web Application:
```bash
cd guacamole
wget https://archive.apache.org/dist/guacamole/1.5.4/binary/guacamole-1.5.4.war
mv guacamole-1.5.4.war /var/lib/tomcat9/webapps/guacamole.war
```

### Set Up Environment Variables:
Edit `/etc/default/tomcat`:
```bash
vim /etc/default/tomcat
```
Add:
```bash
GUACAMOLE_HOME=/etc/guacamole
```

Edit `/etc/profile`:
```bash
vim /etc/profile
```
Add:
```bash
export GUACAMOLE_HOME=/etc/guacamole
```

### Configure guacamole.properties:
Create and edit the configuration file:
```bash
vim /etc/guacamole/guacamole.properties
```
Add:
```
guacd-hostname: 127.0.0.1
guacd-port: 4822
```

```bash
systemctl daemon-reload
```

Restart services:
```bash
sudo systemctl restart tomcat9 guacd
```

---

## Step 5: Setup database Authentication

Install MySQL:
```bash
sudo apt update
sudo apt upgrade
sudo apt install mysql-server
sudo systemctl start mysql.service
```
Now log in to the MySQL console
 ```bash
mysql -uroot -p
 ```

Create a database and user:
```sql
CREATE DATABASE guacamole_db;
CREATE USER 'guacamole_user'@'localhost' IDENTIFIED BY 'Abcd@123';
GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD on *.* TO 'guacamole_user'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

Download and configure the MySQL extension:
```bash
cd guacamole
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-8.3.0.tar.gz
tar -xf mysql-connector-j-8.3.0.tar.gz
cp mysql-connector-j-8.3.0/mysql-connector-j-8.3.0.jar /etc/guacamole/lib/

wget https://downloads.apache.org/guacamole/1.5.4/binary/guacamole-auth-jdbc-1.5.4.tar.gz
tar -xf guacamole-auth-jdbc-1.5.4.tar.gz
mv guacamole-auth-jdbc-1.5.4/mysql/guacamole-auth-jdbc-mysql-1.5.4.jar /etc/guacamole/extensions/
```

Import SQL schema:
```bash
cd guacamole-auth-jdbc-1.5.4/mysql/schema
cat *.sql | mysql -u root -p guacamole_db
```

Update `guacamole.properties`:
```bash
vim /etc/guacamole/guacamole.properties
```
Add:
```
mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guac_user
mysql-password: guac_pass
```
```bash
systemctl daemon-reload
```

Restart services:
```bash
sudo systemctl restart tomcat9 guacd mysql
```

---

## Step 6: Access Guacamole Web Interface

Open your browser and navigate to:
```
http://<your-public-ip>:8080/guacamole
```
![alt text](<WhatsApp Image 2024-12-26 at 15.31.41.jpeg>)


---

## Step 7: Configure VNC for Android Remote Access

### Install VNC Server on Android
I install droidVNC-NG application  and after that set configuration 

![alt text](<droidVNC-NG.jpeg>)
 and go to the 
 
 ```
http://<your-ip-or-domain>:8080/guacamole
```
create a connection go to the setting->connection->Newconnection
![alt text](<new_connection.jpeg>)


