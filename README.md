# Nginx-setup-and-hosting-laravel-apps-on-VPS

---

title: Setting Up Laravel in Ubuntu / DigitalOcean
keywords: servers, laravel, coderstape, coder's tape
description: Let's take a look at settting up a server from scratch for Laravel.
date: May 13, 2021
tags: servers, laravel

---

## Getting Started

- Create droplet with Ubuntu 18.10
- `ssh root@[DROPLET IP ADDRESS]`
- Get password from your email
- Change password on first login
- `adduser laravel`
- Enter password and other information
- `usermod -aG sudo laravel`

## Locking Down to SSH Key only (Extremely Important)

- In your local machine, `ssh-keygen`
- Generate a key, if you leave passphrase blank, no need for password
- `ls ~/.ssh` to show files in local machine
- Get the public key, `cat ~/.ssh/id_rsa.pub`
- Copy it
- `cd ~/.ssh` and `vim authorized_keys`
- Paste key
- Repeat steps for laravel user
- `su laravel` then `mkdir ~/.ssh` fix permissions `chmod 700 ~/.ssh`
- `vim ~/.ssh/authorized_keys` and paste key
- `chmod 600 ~/.ssh/authorized_keys` to restrict this from being modified
- `exit` to return to root user

## Disable Password from Server

- `sudo vim /etc/ssh/sshd_config`
- Find PasswordAuthentication and set that to `no`
- Turn on `PubkeyAuthentication yes`
- Turn off `ChallengeResponseAuthentication no`
- Reload the SSH service `sudo systemctl reload sshd`
- Test new user in a new tab to prevent getting locked out

## Setting Up Firewall

- View all available firewall settings
- `sudo ufw app list`
- Allow on OpenSSH so we don't get locked out
- `sudo ufw allow OpenSSH`
- Enable Firewall
- `sudo ufw enable`
- Check the status
- `sudo ufw status`

## Install Linux, Nginx, MySQL, PHP

### Nginx

- `sudo apt update` enter root password
- `sudo apt install nginx` enter Y to install
- `sudo ufw app list` For firewall
- `sudo ufw allow 'Nginx HTTP'` to add NGINX
- `sudo ufw status` to verify change
- Visit server in browser

### MySQL

- `sudo apt install mysql-server` enter Y to install
- `sudo mysql_secure_installation` to run automated securing script
- Press N for VALIDATE PASSWORD plugin
- Set root password
- Remove anonymous users? `Y`
- Disallow root login remotely? `N`
- Remove test database and access to it? `Y`
- Reload privilege tables now? `Y`
- `sudo mysql` to enter MySQL CLI
- `SELECT user,authentication_string,plugin,host FROM mysql.user;` to verify root user's auth method
- `ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'STRONG_PASSWORD_HERE';` to set a root password
- `SELECT user,authentication_string,plugin,host FROM mysql.user;` to verify root user's auth method
- `FLUSH PRIVILEGES;` to apply all changes
- `mysql -u root -p` to access db from now on, enter password `STRONG_PASSWORD_HERE`

### PHP & Basic Nginx

- `sudo add-apt-repository universe` to add software repo
- `sudo apt install php-fpm php-mysql` to install the basic PHP software
- `sudo vim /etc/nginx/sites-available/YOUR.DOMAIN.COM`

```
server {
        listen 80;
        root /var/www/html;
        index index.php index.html index.htm index.nginx-debian.html;
        server_name YOUR.DOMAIN.COM;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        }

        location ~ /\.ht {
                deny all;
        }
}
```

- `sudo ln -s /etc/nginx/sites-available/YOUR.DOMAIN.COM /etc/nginx/sites-enabled/` to create symlink to enabled sites
- `sudo unlink /etc/nginx/sites-enabled/default` to remove default link
- `sudo nginx -t` test the whole config
- `sudo systemctl reload nginx` to apply all changes
- `sudo vim /var/www/html/info.php` to start a new PHP file, fill it with <?php phpinfo();
- `sudo rm /var/www/html/info.php` optional command to get rid of test file

## Let's Dial in The Laravel Ecosystem

- `sudo apt-get install php7.2-mbstring php7.2-xml composer unzip`
- `mysql -u root -p` Login to create the Laravel DB
- `CREATE DATABASE laravel DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;`
- `GRANT ALL ON laravel.* TO 'laraveluser'@'localhost' IDENTIFIED BY 'password';`
- `FLUSH PRIVILEGES;`
- `exit`
- `cd /var/www/html`, `sudo mkdir -p first-project`
- `sudo chown laravel:laravel first-project`
- `git clone https://github.com/coderstape/laravel-58-from-scratch.git .`
- `composer install`
- `cp .env.example .env`, and then `vim .env`

```
APP_NAME=Laravel
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=http://YOUR.DOMAIN.COM

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=root
DB_USERNAME=laravel
DB_PASSWORD=STRONG_PASSWORD_HERE
```

- `php artisan migrate`
- `php artisan key:generate` to generate the key
- `sudo chgrp -R www-data storage bootstrap/cache` fix permissions
- `sudo chmod -R ug+rwx storage bootstrap/cache` fix permissions
- `sudo chmod -R 755 /var/www/html/first-project` fix permissions
- `chmod -R o+w /var/www/html/first-project/storage/` fix permission

## Modify Nginx

- `sudo vim /etc/nginx/sites-available/YOUR.DOMAIN.COM`

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/html/first-project/public;
    index index.php index.html index.htm index.nginx-debian.html;

    server_name YOUR.DOMAIN.COM;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }

    location ~ /\.ht {
            deny all;
    }
}
```

- `sudo nginx -t`
- `sudo systemctl reload nginx` reload Nginx

## Let's Encrypt

- `sudo add-apt-repository ppa:certbot/certbot` to get repo
- `sudo apt install python-certbot-nginx` to install
- `sudo certbot certonly --webroot --webroot-path=/var/www/html/quickstart/public -d example.com -d www.example.com`
- `sudo certbot certonly --webroot --webroot-path=/var/www/html/first-project/public -d YOUR.DOMAIN.COM`

### Final mod for Nginx

- `sudo vim /etc/nginx/sites-available/YOUR.DOMAIN.COM`

```
server {
    listen 80;
    listen [::]:80;

    server_name YOUR.DOMAIN.COM;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name YOUR.DOMAIN.COM;
    root /var/www/html/first-project/public;

    ssl_certificate /etc/letsencrypt/live/YOUR.DOMAIN.COM/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/YOUR.DOMAIN.COM/privkey.pem;

    ssl_protocols TLSv1.2;
	ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
	ssl_prefer_server_ciphers on;

	add_header X-Frame-Options "SAMEORIGIN";
	add_header X-XSS-Protection "1; mode=block";
	add_header X-Content-Type-Options "nosniff";

	index index.php index.html index.htm index.nginx-debian.html;

    charset utf-8;

    location / {
            try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }

    location ~ /\.ht {
            deny all;
    }

    location ~ /.well-known {
            allow all;
    }
}
```

- `sudo nginx -t`
- `sudo ufw app list` For firewall
- `sudo ufw allow 'Nginx HTTPS'` to add NGINX
- `sudo ufw status` to verify change
- `sudo systemctl reload nginx` reload Nginx

## Extra Credit

Let's make the prompt pretty

- `sudo apt-get install zsh` to install ZSH
- `zsh --version` to confirm install
- `whereis zsh` to find out where it is
- `sudo usermod -s /usr/bin/zsh $(whoami)` to make Zsh default
- `sudo reboot` to reapply all changes
- `2` to populate a default file
- `sudo apt-get install powerline fonts-powerline` to install powerline
- `sudo apt-get install zsh-theme-powerlevel9k` to install Theme
- `echo "source /usr/share/powerlevel9k/powerlevel9k.zsh-theme" >> ~/.zshrc` to enable the theme in your Zshrc
- `exit` and login again to see the new theme
- `sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"` for Oh My Zsh
- `echo "source /usr/share/powerlevel9k/powerlevel9k.zsh-theme" >> ~/.zshrc` to re-enable 9K
