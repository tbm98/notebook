# Install Nginx and Let’s Encrypt with Docker – Ubuntu 20.04

Source: [https://www.cloudbooklet.com/how-to-install-nginx-and-lets-encrypt-with-docker-ubuntu-20-04/](https://www.cloudbooklet.com/how-to-install-nginx-and-lets-encrypt-with-docker-ubuntu-20-04/)

## Create Docker Compose YML file

Now SSH inside your server or Virtual machine and create a directory to hold all the configurations by running the following command.

```
sudo mkdir ~/nginx-ssl
```

Move inside the directory and create a **docker-compose.yml** file that holds our configuration.

```
cd ~/nginx-ssl
sudo nano ~/nginx-ssl/docker-compose.yml
```

Paste the following configurations inside the file.

```
version: "3.8"
services:
    web: 
        image: nginx:latest
        restart: always
        volumes:
            - ./public:/var/www/html
            - ./conf.d:/etc/nginx/conf.d
            - ./certbot/conf:/etc/nginx/ssl
            - ./certbot/data:/var/www/certbot
        ports:
            - 80:80
            - 443:443

    certbot:
        image: certbot/certbot:latest
        command: certonly --webroot --webroot-path=/var/www/certbot --email your-email@domain.com --agree-tos --no-eff-email -d domain.com -d www.domain.com
        volumes:
            - ./certbot/conf:/etc/letsencrypt
            - ./certbot/logs:/var/log/letsencrypt
            - ./certbot/data:/var/www/certbot
```

Hit **CTRL-X** followed by **Y** and **ENTER** to save and exit the file.

Here are the configuration details.

* **version**: Compose file version which is compatible with the Docker Engine. You can check compatibility [`here`](https://docs.docker.com/compose/compose-file/).
* **image**: We use latest [Nginx](https://hub.docker.com/\_/nginx?tab=tags) and [Certbot](https://hub.docker.com/r/certbot/certbot/tags) images available in Docker hub.
* **volumes**:
  * _public_: we have configured this directory to be synced with the directory we wish to use as the web root inside the container.
  * _conf.d_: here we will place the Nginx configuration file to be synced with the default Nginx conf.d folder inside the container.
  * _certbot/conf_: this is where we will receive the SSL certificate and this will be synced with the folder we wish to inside the container.
  * _ports_: configure the container to listen upon the listed ports.
  * _command_: the command used to receive the SSL certificate.

Now you have your **docker-compose.yml** in place.

## Configure Nginx

Now we need to configure Nginx for validation to obtain the Let’s Encrypt SSL certificate.

We will create a directory as mentioned in the docker-compose file as **conf.d**.

```
sudo mkdir ~/nginx-ssl/conf.d
```

Create a configuration file with the .conf extension.

```
sudo nano ~/nginx-ssl/conf.d/default.conf
```

Paste the following configuration inside the file.

```
server {
     listen [::]:80;
     listen 80;

     server_name domain.com www.domain.com;

     location ~ /.well-known/acme-challenge {
         allow all; 
         root /var/www/certbot;
     }
} 
```

Hit **CTRL-X** followed by **Y** and **ENTER** to save and exit the file.

Now you have the Nginx configuration which gets synced to the **/etc/nginx/conf.d** folder which automatically gets loaded by Nginx.

## Start Containers

Now it’s time to start the containers using the following command to receive the SSL certificates.

You need to pass the -d flag which starts the container in background and leaves them running.

```
docker-compose up -d
```

You will see an output similar to the one below.

```
Output
Creating network "nginx-ssl_default" with the default driver
Pulling web (nginx:latest)…
latest: Pulling from library/nginx
8559a31e96f4: Pull complete
8d69e59170f7: Pull complete
3f9f1ec1d262: Pull complete
d1f5ff4f210d: Pull complete
1e22bfa8652e: Pull complete
Digest: sha256:21f32f6c08406306d822a0e6e8b7dc81f53f336570e852e25fbe1e3e3d0d0133
Status: Downloaded newer image for nginx:latest
Pulling certbot (certbot/certbot:latest)…
latest: Pulling from certbot/certbot
cbdbe7a5bc2a: Pull complete
26ebcd19a4e3: Pull complete
a29d43ca1bb4: Pull complete
979dbbcf63e0: Pull complete
30beed04940c: Pull complete
48a1f8a4d505: Pull complete
4416e9b4bbe0: Pull complete
8173b4be7870: Pull complete
21c8dd124dab: Pull complete
c19b04e11dc7: Pull complete
1b560611cec1: Pull complete
Digest: sha256:568b8ebd95641a365a433da4437460e69fb279f6c9a159321988d413c6cde0ba
Status: Downloaded newer image for certbot/certbot:latest
Creating nginx-ssl_certbot_1 … done
Creating nginx-ssl_web_1     … done
```

This output indicates Nginx and Certbot images are pulled from Docker hub and the containers are created successfully.

To view the containers you can execute the following command.

```
docker-compose ps
```

```
Output
       Name                      Command               State                     Ports
nginx-ssl_certbot_1   certbot certonly --webroot …   Exit 0                                           
nginx-ssl_web_1       /docker-entrypoint.sh ngin …   Up       0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
```

The state **Exit 0** indicates the setup is completed without any error.

Now when you check your work directory, there will be a new directory created as **certbot** inside which you will have the SSL certificate synced.

```
ls ~/nginx-ssl/certbot/conf/live/domain.com
```

## Configure SSL with Nginx

As you have received the Let’s Encrypt SSL certificate you can configure HTTPS and setup redirection to HTTPS.

Edit the **default.conf** and make the following changes.

Your file should like the one below at the final stage.

```
server {
    listen [::]:80;
    listen 80;

    server_name domain.com www.domain.com;

    location ~ /.well-known/acme-challenge {
        allow all; 
        root /var/www/certbot;
    }

    # redirect http to https www
    return 301 https://www.domain.com$request_uri;
}

server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    server_name domain.com;

    # SSL code
    ssl_certificate /etc/nginx/ssl/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/domain.com/privkey.pem;

    root /var/www/html;

    location / {
        index index.html;
    }

    return 301 https://www.domain.com$request_uri;
}

server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    server_name www.domain.com;

    # SSL code
    ssl_certificate /etc/nginx/ssl/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/domain.com/privkey.pem;

    root /var/www/html/domain-name/public;

    location / {
        index index.html;
    }
} 
```

Hit **CTRL-X** followed by **Y** and **ENTER** to save and exit the file.

## Create index.html file

Now you can create the index.html file inside the public directory which then syncs to the directory configured.

Create the public directory.

```
sudo mkdir ~/nginx-ssl/public
```

```
sudo nano ~/nginx-ssl/public/index.html
```

```
<html>
    <body>
        <h1>Docker setup with Nginx and Let's Encrypt SSL.</h1>
    </body>
</html
```

Hit **`CTRL-X`** followed by **`Y`** and **`ENTER`** to save and exit the file.

## Restart the containers&#x20;

Now you can restart the containers to load the new configurations.

```
docker-compose restart 
```

Once the containers are restarted you can check your domain name. You will get a redirection to HTTPS and your SSL.
