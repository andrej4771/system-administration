
# Network Services Lab – System Administration

This repository contains **step-by-step documentation** and **verified commands** for configuring core network services on **Ubuntu Linux** as part of the ANS lab exercises.

> Focus: correctness, security best practices, and exam-ready explanations.

---

## Contents

1. User & Group Management + Hostname
2. DNS Server (BIND9)
3. Apache Web Server
4. Moodle (Apache + PHP + MySQL) on Port 8088
5. WordPress (Apache VirtualHost)
6. FTP Server (vsftpd)
7. Firewall (UFW + iptables)
8. Apache Reverse Proxy for Node.js

---

## 1. User & Group Management + Hostname

### Goal

* Create user `ansXXXXXX`
* Create group `ans` and add the user to it
* Set hostname to `ansDDMMYYYY`

### Commands

```bash
sudo adduser ans232072
sudo groupadd ans
sudo usermod -aG ans ans232072
groups ans232072
sudo hostnamectl set-hostname ans27012005
```

---

## 2. DNS Server – BIND9

### Goal

* Install BIND9
* Configure local DNS resolver
* Allow queries from `10.10.0.0/16`
* Create a **master zone** `232072.io.mk`
* Configure forwarders

### Installation

```bash
sudo apt update
sudo apt install bind9 -y
```

### Local Resolver

```bash
sudo nano /etc/resolv.conf
# nameserver 127.0.0.1
```

### BIND Options

```bash
sudo nano /etc/bind/named.conf.options
```

```conf
allow-query { 10.10.0.0/16; localhost; };
listen-on { any; };
forwarders { 1.1.1.1; 1.0.0.1; };
```

```bash
sudo named-checkconf
sudo systemctl restart bind9
```

### Zone Definition

```bash
sudo nano /etc/bind/named.conf.local
```

```conf
zone "232072.io.mk" {
    type master;
    file "/etc/bind/zones/db.232072.io.mk";
};
```

### Zone File

```bash
sudo mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/db.232072.io.mk
```

```dns
$TTL 86400
@   IN  SOA ns1.232072.io.mk. admin.232072.io.mk. (
        2024012701
        3600
        1800
        604800
        86400 )

    IN  NS  ns1.232072.io.mk.
    IN  NS  ns.ans.mk.

ns1     IN  A   10.155.2.113
www     IN  A   10.155.2.113
ftp2    IN  A   10.155.2.113
dns     IN  A   10.10.16.200

web     IN  CNAME www.232072.io.mk.
```

### Testing

```bash
dig @localhost www.232072.io.mk
dig @localhost web.232072.io.mk
dig @localhost dns.232072.io.mk
```

---

## 3. Apache Web Server

### Definition

Apache HTTP Server is a widely used open-source web server that listens for HTTP requests and serves web content such as HTML, PHP applications, and static files.

### Installation

```bash
sudo apt update                  # Updates package lists
sudo apt install apache2 -y       # Installs Apache web server
systemctl status apache2          # Verifies that Apache is running
```

Apache listens on TCP port **80** by default and serves files from `/var/www/html` unless virtual hosts are configured.

---

## 4. Moodle (Apache + PHP + MySQL) on Port 8088

### Definition

Moodle is an open-source Learning Management System (LMS) written in PHP. It requires a web server (Apache), a database server (MySQL/MariaDB), and PHP with required extensions.

This task installs Moodle as a **PHP application served by Apache**, listening on **port 8088**.

### PHP Installation

```bash
sudo apt install php libapache2-mod-php -y   # Installs PHP and Apache PHP module
php -v                                      # Verifies PHP installation
```

### Configure Apache to Listen on Port 8088

```bash
sudo nano /etc/apache2/ports.conf            # Apache listens on additional port
# Add the following line below Listen 80:
Listen 8088
```

### Database Setup (MySQL)

```bash
sudo apt install mysql-server -y             # Installs MySQL server
sudo mysql                                   # Opens MySQL shell
```

```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4;   -- Moodle database
CREATE USER 'moodleuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodleuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Moodle Source Code Installation

```bash
sudo apt install git -y                      # Installs Git
cd /var/www/html
sudo git clone git://git.moodle.org/moodle.git   # Clones Moodle repository
cd moodle
sudo git checkout MOODLE_501_STABLE           # Switches to stable Moodle version
```

### Moodle Data Directory

```bash
sudo mkdir /var/moodledata                   # Directory for user data
sudo chown -R www-data:www-data /var/www/html/moodle /var/moodledata
sudo chmod -R 755 /var/www/html/moodle
sudo chmod -R 770 /var/moodledata
```

### Apache VirtualHost for Moodle

```bash
sudo nano /etc/apache2/sites-available/moodle.conf
```

```apache
<VirtualHost *:8088>
    DocumentRoot /var/www/html/moodle         # Moodle application root

    <Directory /var/www/html/moodle>
        AllowOverride All                     # Allows .htaccess overrides
        Require all granted                   # Public access
    </Directory>
</VirtualHost>
```

```bash
sudo a2ensite moodle.conf                     # Enables Moodle site
sudo a2enmod rewrite                          # Enables URL rewriting
sudo systemctl reload apache2
```

Access Moodle via:

```
http://<SERVER_IP>:8088
```

---

## 5. WordPress (Apache + PHP + MySQL)

### Definition

WordPress is an open-source Content Management System (CMS) written in PHP. It requires Apache, PHP extensions, and a MySQL database.

### Required Packages

```bash
sudo apt install php php-mysql php-curl php-gd php-mbstring php-xml php-zip -y
sudo apt install mysql-server -y
```

### Database Setup

```sql
CREATE DATABASE wordpress;                    -- WordPress database
CREATE USER wordpress@localhost IDENTIFIED BY '123';
GRANT ALL PRIVILEGES ON wordpress.* TO wordpress@localhost;
FLUSH PRIVILEGES;
```

### WordPress Files

```bash
sudo mkdir -p /srv/www                       # Web root directory
sudo chown www-data: /srv/www                # Apache ownership
curl https://wordpress.org/latest.tar.gz | sudo -u www-data tar zx -C /srv/www
```

### Apache VirtualHost

```apache
<VirtualHost *:80>
    ServerName wp.index.com                  # Domain name
    DocumentRoot /srv/www/wordpress

    <Directory /srv/www/wordpress>
        AllowOverride All                    # Enables .htaccess
        Require all granted                  # Public access
    </Directory>
</VirtualHost>
```

---

## 6. NFS Client Configuration

### Goal

* Install NFS client utilities
* Create mount points `/nfs/jan2024` and `/nfs/proba2024`
* Mount remote NFS exports from server `10.10.16.200`
* Configure persistent mounts using `/etc/fstab`
* Create a directory with index number and a test file

### Installation

```bash
sudo apt update
sudo apt install nfs-common -y
```

### Create Mount Directories

```bash
sudo mkdir -p /nfs/jan2024
sudo mkdir -p /nfs/proba2024
ls -ld /nfs/jan2024 /nfs/proba2024
```

### Mount NFS Directories

```bash
sudo mount 10.10.16.200:/mnt/kolokvium_2024 /nfs/jan2024
sudo mount 10.10.16.200:/mnt/proba /nfs/proba2024
```

### Persistent Mounts (/etc/fstab)

```bash
sudo nano /etc/fstab
```

```fstab
10.10.16.200:/mnt/kolokvium_2024  /nfs/jan2024    nfs    defaults    0   0
10.10.16.200:/mnt/proba           /nfs/proba2024  nfs    defaults    0   0
```

### Test Mounts

```bash
sudo umount /nfs/jan2024
sudo umount /nfs/proba2024
sudo mount -a
```

### Create Index Directory and File

```bash
cd /nfs/jan2024
mkdir 232072
cd 232072
touch test.txt
ls -l
```

---

## 7. FTP Server (vsftpd)

### Installation

```bash
sudo apt install vsftpd -y
```

### User & Directories

```bash
sudo adduser editor
sudo mkdir -p /home/student/ftp
sudo chown editor:editor /home/student/ftp
sudo chmod a-w /home/student/ftp
sudo mkdir /home/student/ftp/files
sudo chown editor:editor /home/student/ftp/files
```

### Configuration

```conf
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
user_sub_token=$USER
local_root=/home/student/ftp
```

### Test

```bash
echo "TEST" > test.txt
ftp <SERVER_IP>
```

---

## 7. Firewall (UFW + iptables)

### UFW

```bash
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:50000/tcp
```

### iptables (Exam Reference)

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A INPUT -p tcp --dport 40000:50000 -j ACCEPT
```

---

## 8. Apache Reverse Proxy for Node.js

### Node.js App

```bash
sudo apt install nodejs npm -y
mkdir ~/nodeapp && cd ~/nodeapp
nano app.js
```

```js
const http = require("http");
http.createServer((req, res) => {
  res.end("Node.js behind Apache proxy\n");
}).listen(3000);
```

### Apache Proxy

```bash
sudo a2enmod proxy proxy_http
```

```apache
<VirtualHost *:80>
    ServerName node.index.com
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/
</VirtualHost>
```

---

## License

Educational use only.
