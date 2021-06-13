# Deploying Django Project to Digital Ocean

We host single Django Project on Digital Ocean. The following steps should work on any cloud Iaas.

### 1. Create ssh key and add to DO
```commandline
$ ssh-keygen -f KEYNAME -C noname
```
### 2. Choose droplet deploy with SSH key
Setup a droplet, using Ubuntu 20.04 (LTS) x64.
### 3. Connect to your droplet
```commandline
$ ssh -i "YOURPRIVATEKEYNAME" ROOT@SERVERIP
```
### 4. Update packages
```commandline
$ sudo apt-get update
```
### 5. Add more users
```commandline
$ sudo adduser USERNAME
```
### 6. Make user sudo if needed
```commandline
$ sudo usermod -aG sudo USERNAME
```
### 7. Add users public key
```commandline
$ su USERNAME
$ mkdir ~/.ssh/
$ cd
$ nano .ssh/authorized_keys
```
Now paste in their public key.

### 8. Harden security
```commandline
$ nano /etc/ssh/sshd_config
```

Then set the following

```
PermitRootLogin no
StrictModes yes
PubkeyAuthentication yes
PasswordAuthentication no
```

### 9. Now restart SSH
```commandline
$ sudo service ssh restart
```

### 10. Set up Firewall rules

All non-essential ports should be blocked using ufw.
First switch off ufw (to ensure we don't kick ourselves off server when blocking all ports):

```commandline
$ sudo ufw disable
``` 

The open required ports:

SSH ```$ sudo ufw allow 22```
HTTP ```$ sudo ufw allow 80```
HTTPS ```$ sudo ufw allow 443```

To ensure config is saved when changed install iptables-persistent:

```commandline
$ sudo apt-get install iptables-persistent
```

And save iptables set above

```commandline
$ sudo service netfilter-persistent save
```

Then turn on ufw to enable rules: ```$ sudo ufw enable```

### 11. Deploy app to server

You can

1. set up auto deployments. [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-automatic-deployment-with-git-with-a-vps)

2. Manually clone from the repository: 
```commandline
$ git clone https://github.com/$target-repo
```

### 12. Install PostgreSQL

Docs here: [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-20-04)

```commandline
sudo apt install postgresql postgresql-contrib;

sudo -i -u postgres;

psql;

ALTER USER postgres PASSWORD 'myPassword';

exit;

sudo apt install postgis postgresql-12-postgis-3
```

### 13. Install Python Packages

```commandline
sudo -H pip3 install virtualenv

virtualenv venv

source venv/bin/activate

pip install -r requirements.txt
```

### 15. Install DB tables
```commandline
python manage.py migrate
```

### 16. Create first staff user for django admin

```commandline
python manage.py createsuperuser
```

### 17. exit venv

```commandline
deactivate
```

### 18. Install Nginx

```commandline
apt-get update
apt-get install nginx -y
systemctl enable nginx
systemctl start nginx
```

create nginx config file.

```commandline
sudo nano /root/myproject.conf_bak
sudo nano /etc/nginx/sites-enabled/myproject.conf
```

fill the files following contents.

```.editorconfig
server {
    server_name $YOUR_DOMAIN_NAME;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
         alias $app_path/staticfiles/;
    }

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:8000/;
    }
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/staging.mapthepaths.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/$YOUR_DOMAIN_NAME/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    client_max_body_size 20M;
}
server {
    if ($host = $YOUR_DOMAIN_NAME) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    listen 80;
    server_name $YOUR_DOMAIN_NAME;
    return 404; # managed by Certbot
}
```

* For static files on production server

```
location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        proxy_pass http://app_server;
        break;
    }
```

### 19. Install Encrypt SSL

```commandline
snap install core 
sudo snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot --nginx

certbot --nginx  --agree-tos --register-unsafely-without-email -d $YOUR_DOMAIN_NAME www.$YOUR_DOMAIN_NAME
```

to check
```commandline
cd /etc/letsencrypt/live
cd $YOUR_DOMAIN_NAME
cat cert.pem
```

### 20 Create Python Service File

```commandline
sudo nano /etc/systemd/system/python.service
```

fill the file following content.

```editorconfig
[Unit]
Description=Python daemon
After=network.target

[Service]
User=root
Group=root
WorkingDirectory=$APP_ROOT_PATH
ExecStart=$APP_ROOT_PATH/venv/bin/python3 $APP_ROOT_PATH/manage.py runserver

[Install]
WantedBy=multi-user.target
```

### 21 Run Django APP

```commandline
systemctl daemon-reload
systemctl start python.service
systemctl enable python.service
```

Check Django Service

```commandline
systemctl status python.service
```

Stop Django Service

```commandline
systemctl stop python.service
```

Restart MTPW Service

```commandline
systemctl restart python.service
```
