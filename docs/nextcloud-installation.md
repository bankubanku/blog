I needed a cloud storage and rather than use Google Drive, Dropbox or similar service I decided to utilize an old laptop I got from my parents to install [Nextcloud](https://nextcloud.com/) on it, and learn about the process of doing it. This post will describe steps I've taken to make it work.  

I operated on [Ubuntu 22.04 LTS](https://ubuntu.com/download/server) which is recommended by Nextcloud's [docs](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html), chose [Apache](https://apache.org/) as a web server and [MariaDB](https://mariadb.com) as a database. 

I updated the system and installed required packages like apache2, mariadb-server which I mentioned earlier, libapache2-mod-php - a library that handles communication between Apache and [PHP](https://www.php.net/). The rest of the packages are PHP modules that Nextcloud needs to work properly.   

```bash
sudo apt update && sudo apt upgrade
```

```shell
sudo apt install apache2 mariadb-server libapache2-mod-php php-gd php-mysql php-curl php-mbstring php-intl php-gmp php-bcmath php-xml php-imagick php-zip
```

# database
I started MySQL by simply typing 

```shell
sudo mysql
```

Then I entered a few lines. 

```MySQL
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
```

The first line will create a user with a name of your choice (replace `username` with that name) on a local database that will be authenticated with password (replace `password` with your own, strong password). 

The second line will create a database called `nextcloud` (if it doesn't exist) that will store data using 4-Byte UTF-8 Unicode Encoding and set up [collation](https://dev.mysql.com/doc/refman/8.0/en/charset-general.html) to recommended by official Nextcloud's [docs](https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html#) `utf8mb4_general_ci`.   

Next line grants all privileges to the user created by me earlier that will apply on the created database. 

Last line reload granted privileges into a memory. Although I'm not sure if that's necessary, because MySQL's [docs](https://dev.mysql.com/doc/refman/8.0/en/privilege-changes.html) says that it happens automatically when granting these privileges. I decided to do this anyway, because Nextcloud's included it in its [Example installation on Ubuntu 22.04 LTS](https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html). 

That's all that had to be done with the database. 

```SQL
quit;
```

# download nextcloud 
To do that I went to [Nextcloud Download Page](https://nextcloud.com/install), scrolled down to the "Download Server" section, expanded the "Community Projects" drop-down, copied link to a zip archive (you can choose an archive with .tar.bz2 extension) and downloaded it using wget 

```shell
wget https://download.nextcloud.com/server/releases/latest.zip
```

To verify if the archive is not corrupted, I downloaded and checked MD5 sum. 

```shell
wget https://download.nextcloud.com/server/releases/latest.zip.md5
```

```shell
md5sum -c latest.zip.md5 < latest.zip
```

I extracted the archive,

```shell
unzip latest.zip
```

copied it into `/var/www`

```shell
sudo cp -r nextcloud /var/www
```

and changed the ownership to HTTP user.

```shell
sudo chown -R www-data:www-data /var/www/nextcloud
```

# apache
I created a configuration file in `/etc/apache2/sites-available/nextcloud.conf` and filled out with the following content.

```
<VirtualHost *:80>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.myserver.tech

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
```

I chose virtual host installation, which makes Nextcloud accessible from its own subdomain (i.e. cloud.myserver.tech) in opposition to directory-based installation (i.e. myserver.tech\/nextcloud). I copied and pasted this configuration from Nextcloud's docs and changed only the `ServerName` to my own server name. 

To enable this configuration, I typed the following command. 

```shell
sudo a2ensite nextcloud.conf
```

Then I made sure that all required and recommended modules are enabled.

```shell
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
```

Last thing I did was restart Apache service to apply all changes.  

```shell
sudo systemctl restart apache2
```

# installation wizard 
To connect and finish the installation, I added a line to `/etc/hosts` on a machine from which I will do that.

```
192.168.1.22  cloud.myserver.tech  cloud
```

Now, when I type `cloud.myserver.tech` in my browser, it will show me a graphical interface to install Nextcloud. I mean it should, but it didn't. The error occurred.  
I checked `/var/log/apache2/error.log` and I saw a following message. 

```
Fri Jul 14 13:44:58.871056 2023] [php:error] [pid 910] [client 192.168.1.4:34214] PHP Fatal error:  Uncaught Error: Class "RedisCluster" not found in /var/www/nextcloud/config/config.php:1472\nStack trace:\n#0 /var/www/nextcloud/lib/private/Config.php(233): include()\n#1 /var/www/nextcloud/lib/private/Config.php(71): OC\\Config->readData()\n#2 /var/www/nextcloud/lib/base.php(151): OC\\Config->__construct()\n#3 /var/www/nextcloud/lib/base.php(612): OC::initPaths()\n#4 /var/www/nextcloud/lib/base.php(1173): OC::init()\n#5 /var/www/nextcloud/index.php(34): require_once('...')\n#6 {main}\n  thrown in /var/www/nextcloud/config/config.php on line 1472
```

After some research I concluded that this error happened, because a PHP module called `php-redis` wasn't installed, so I installed it and restarted apache. A similar error occurred right after, so I did the same. 

Everything was alright after those fixes and I finished installation in graphical interface. I filled out the form with my username and password that I wanted to log in to my file server, changed the path to the database (it's recommended by the docs), and typed the rest of the information about the database I created earlier. I clicked install.

![[../images/installation-wizard.png]]

Another error occurred.  

![[../images/installation-wizard-error.png]]

I forgot to create  `/opt/nextcloud/data` and changed its owner to `www-data`.  Quickly did that and clicked "install" again.

```shell
sudo mkdir -p /opt/nextcloud/data
sudo chown www-data:www-data /opt/nextcloud/data
```

I preferred to not install recommended apps for now, so I skipped this stage. 

![[../images/installation-wizard-recommended-apps.png]]

# access via the internet 
At this point, Nextcloud is installed, but it could be accessed only via my local network. 

To use my file server from any place in the world, I needed DDNS (Dynamic DNS). I decided to use [DuckDNS](https://duckdns.org). It is pretty simple to set up with its guide and interface, so I won't elaborate how I did it exactly. I just logged in with my GitHub account, added a domain with a name I wanted, and went to an "install" section to finish setting up. 

In my router settings I bound an IP address to my server, so DHCP wouldn't change that, and forwarded port 80 (HTTP) and 443 (HTTPS) to this IP address. 

# ssl certificate 
First, I installed two packages.

```shell
sudo apt install certbot python3-certbot-apache
```

Then I typed the command below and provided the needed information. 

```shell
sudo certbot --apache -d cloud.myserver.tech
```

That's all, SSL is set up. [Certbot](https://certbot.eff.org) modified Apache's configuration files to work with HTTPS, so I didn't have to do it myself. 

# summary 
Now I can use Nextcloud as a file server. Of course, I will be testing it and installing new apps to enhance my experience, but in this post I wanted only to share the process of installing it and making it usable. 
