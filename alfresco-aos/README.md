# Alfresco AOS Deployment

This project provides Docker Compose template to deploy [AOS](https://docs.alfresco.com/microsoft-office/latest/) Sharepoint Protocol with ACS 7.3

>> This project has been designed to test the integrations easily, deploying them in *prod* environments would require additional operations and resources.


## AOS Sharepoint Prococol

This is a propietary addon, Enterprise customers has access to source code in [Alfresco Nexus Repository](https://nexus.alfresco.com/nexus/service/local/repositories/releases/content/org/alfresco/aos-module/alfresco-aos-module/1.5.0/alfresco-aos-module-1.5.0-sources.jar). The access to this resource requires customer credentials.

This module is provided by default in Alfresco Repository, Alfresco Share and Alfresco Content Application. Since source code is not available for Community users, the feature is available in both ACS Community and Enterprise versions.

The use of AOS requires a trusted TLS certificate. This project provides self-signed certificates available in `config/cert` folder, that may be added as trusted to your computer before running.

Once the module is ready a new action `Edit in Microsoft Office™` is available in Share Web Application and also in Alfresco Content Application.


## Running

Start the Docker Compose using the regular command from project root folder.

```
$ docker compose up
```

Once everything is up & ready, access to following URLs using default credentials (admin/admin):

* Share Web Application https://localhost/share
* Alfresco Content Application https://localhost/


## Configuring Alfresco AOS in EC2 with Let's Encrypt certificate

Following instructions describe how to deploy this Docker Compose in [AWS EC2](https://aws.amazon.com/ec2/) using a trusted *by default* certificate from [Let's Encrypt](https://letsencrypt.org). Using this approach, `Edit in Microsoft Office™` will work without adding manually the certificate in clients.

Configure a new EC2 instance:

* Instance type: `t2.xlarge`
* Platform: `Ubuntu 22.04`
* Security group: Open ports `22`, `80`, `443`

Since [Let's Encrypt](https://letsencrypt.org) is not issuing certificates for default EC2 Public IPv4 DNS, creating a public record with [AWS Route 53](https://aws.amazon.com/route53/) is also required:

* ttl.dev.alfresco.me - CNAME - Simple - ec2-54-75-83-241.eu-west-1.compute.amazonaws.com

>> From this moment, EC2 Public IPv4 DNS `ec2-54-75-83-241.eu-west-1.compute.amazonaws.com` is served using the public DNS `ttl.dev.alfresco.me`

Include this name in the `.env` file in project root folder (`localhost` was used by default):

```
# Server properties
SERVER_NAME=ttl.dev.alfresco.me
```

Prepare NGINX to pass the [Let's Encrypt](https://letsencrypt.org) DNS validation.

```
$ cat config/nginx.conf
worker_processes  1;

events {
    worker_connections  1024;
}

http {

    server {
        listen 80;
        server_name ttl.dev.alfresco.me;
        server_tokens off;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {

        listen *:443 ssl;
...
```

Start Docker Compose from project root folder to listen to DNS validation.

```
$ docker compose up
```    

Generate [Let's Encrypt](https://letsencrypt.org) certificate on EC2 instance.

Install Let's Encrypt client `certbot`:

```
$ sudo apt install certbot python3-certbot-nginx
```

Request a certificate for `ttl.dev.alfresco.me`:

```
$ sudo certbot --nginx --agree-tos --redirect --uir --hsts --staple-ocsp --must-staple -d ttl.dev.alfresco.me --register-unsafely-without-email

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/ttl.dev.alfresco.me/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/ttl.dev.alfresco.me/privkey.pem
```


            - ./config/cert/localhost.cer:/etc/nginx/localhost.cer
            - ./config/cert/localhost.key:/etc/nginx/localhost.key 

Stop Docker Compose:

```
$ docker compose down
```

Apply certificates to NGINX configuration:

```
$ cp /etc/letsencrypt/live/ttl.dev.alfresco.me/fullchain.pem config/cert/
$ cp /etc/letsencrypt/live/ttl.dev.alfresco.me/privkey.pem config/cert/
```

Modify volumes mapping in `docker-compose.yml`:

```
    proxy:
        image: nginx:stable-alpine
        volumes:
            - ./config/cert/fullchain.pem:/etc/nginx/fullchain.pem
            - ./config/cert/privkey.pem:/etc/nginx/privkey.pem
        ports:
            - 443:443
```

Modify NGINX configuration with the new certificate names:

```
$ cat config/nginx.conf
worker_processes  1;

events {
    worker_connections  1024;
}

http {

    server {

        listen *:443 ssl;
        server_name ttl.dev.alfresco.me;
        server_tokens off;

	    ssl_certificate             /etc/nginx/fullchain.pem;
	    ssl_certificate_key         /etc/nginx/privkey.pem;
        ssl_prefer_server_ciphers   on;
        ssl_protocols               TLSv1.2 TLSv1.3;

```

Start again Docker Compose:

```
$ docker compose up 
```

Once everything is up & ready, access to following URLs using default credentials (admin/admin):

* Share Web Application https://ttl.dev.alfresco.me/share
* Alfresco Content Application https://ttl.dev.alfresco.me/

The certificate used for TLS will be the one provided by Let's Encrypt and browsers and applications will trust by default in AOS service.
