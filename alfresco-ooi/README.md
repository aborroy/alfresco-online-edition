# Alfresco Collaboration Connector for Microsoft 365

This project provides Docker Compose template to deploy Office Online Integration (OOI) with ACS 7.3 Enterprise:

* Official documentation available in [Microsoft 365](https://docs.alfresco.com/microsoft-365/latest/)

Docker Images are available in https://quay.io/organization/alfresco, customer credentials are required to use them.

>> This project has been designed to test the integrations easily, deploying them in *prod* environments would require additional operations and resources.


## Configuration

Using OOI requires following configuration elements:

* Alfresco available on a public DNS under a trusted TLS certificate
* Azure Active Directory tenant

>> Before starting this project, review and apply the configuration steps provided in the following points


## ACS Configuration using AWS EC2

OOI requires Alfresco to be installed on a public DNS with a trusted TLS certificate.

Following instructions describe how to deploy this Docker Compose in [AWS EC2](https://aws.amazon.com/ec2/) using a trusted *by default* certificate from [Let's Encrypt](https://letsencrypt.org)

Configure a new EC2 instance:

* Instance type: `t2.xlarge`
* Platform: `Ubuntu 22.04`
* Security group: Open ports `22`, `80`, `443`

Since [Let's Encrypt](https://letsencrypt.org) is not issuing certificates for default EC2 Public IPv4 DNS, creating a public record with [AWS Route 53](https://aws.amazon.com/route53/) is also required:

* ttl.dev.alfresco.me - CNAME - Simple - ec2-54-75-83-241.eu-west-1.compute.amazonaws.com

>> From this moment, EC2 Public IPv4 DNS `ec2-54-75-83-241.eu-west-1.compute.amazonaws.com` is served using the public DNS `ttl.dev.alfresco.me`

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
$ cp /etc/letsencrypt/live/ttl.dev.alfresco.me/fullchain.pem <project-root-folder>
$ cp /etc/letsencrypt/live/ttl.dev.alfresco.me/privkey.pem <project-root-folder>
```

Modify or review volumes mapping in `docker-compose.yml`:

```
    proxy:
        image: nginx:stable-alpine
        volumes:
          - ./nginx.conf:/etc/nginx/nginx.conf
          - ./fullchain.pem:/etc/nginx/fullchain.pem
          - ./privkey.pem:/etc/nginx/privkey.pem
        ports:
            - 443:443
```

Modify or review NGINX configuration with the certificate names:

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

* Alfresco Digital Workspace https://ttl.dev.alfresco.me/workspace/

The certificate used for TLS will be the one provided by Let's Encrypt and browsers and applications will trust by default in OOI service.


## Azure Active Directory tenant configuration

Despite a corporative Azure Active Directory can be used within OOI, following instructions describe the creation of a Tenant for testing purposes.

* Register a new [Microsoft 365 Developer account](https://developer.microsoft.com/en-us/microsoft-365/dev-program) & tenant sandbox
  * note: this may expire (and be deleted) after 90 days so may need to be regularly renewed &/or re-registered
* Login to [Azure Portal](https://portal.azure.com/) as your new tenant admin, eg. myadminuser@mydevdomain.onmicrosoft.com
* Navigate to Azure Active Directory -> App Registrations
  * Register a new app
    * Add a name, eg. Alfresco-Sample-Test
      * note: this name will also appear in the "approot" folder within OneDrive under "/Apps", eg. /Apps/Alfresco-Sample-Test
    * Under Authentication, enable Public Client (edit Manifest json directly and set "allowPublicClient": true)
    * Add API Permissions (for Microsoft Graph) initially Delegated Permissions for each of the following scopes: `User.Read`, `Files.ReadWriteAll`
      * note: this list is subject to change, ie. we will likely need to add a few more API permissions
    * Give Admin Consent for the API permissions
    * Both Admin Consent and setting Public Client above should allow the Sample JUnit tests to now run headless with username/password (ROPC) flow
    * Add in Redirect URIs the Content Services TLS endpoint (for instance https://ttl.dev.alfresco.me) as **Single-page application SPA**

Once everything is configured, application "Alfresco-Sample-Test" will hold values similar to following ones:

```
Display name:            Alfresco-Sample-Test
Application (client) ID: e6738629-4794-47de-86ac-31245842c5c1
Object ID:               d945ff4c-b603-4b0d-aef9-09a25e131021
Directory (tenant) ID:   aa7f6fe7-d6e9-461c-a718-bebc17ead9b1
Supported account types: My organization only
Redirect URIs:           0 web, 1 spa, 0 public client (https://ttl.dev.alfresco.me)
```

Review `docker-compose.yml` file to include the values of your Azure Active Directory.

```
  digital-workspace:
    image: quay.io/alfresco/alfresco-digital-workspace:3.1.1
    mem_limit: 128m
    environment:
      BASE_PATH: ./
      APP_CONFIG_PLUGIN_MICROSOFT_ONLINE: 'true'
      APP_CONFIG_MICROSOFT_ONLINE_OOI_URL: https://ttl.dev.alfresco.me/ooi-service/api/-default-/private/office-integration/versions/1/edit-sessions/
      APP_CONFIG_MICROSOFT_ONLINE_CLIENTID: [Application (client) ID]
      APP_CONFIG_MICROSOFT_ONLINE_AUTHORITY: https://login.microsoftonline.com/[Directory (tenant) ID]
      APP_CONFIG_MICROSOFT_ONLINE_REDIRECT: https://ttl.dev.alfresco.me
```


## Running

Once the TLS and Azure Active Directory is ready, start the Docker Compose using the regular command from project root folder.

```
$ docker compose up
```

Once everything is up & ready, access to following URLs using default credentials (admin/admin):

* Alfresco Digital Workspace https://ttl.dev.alfresco.me/workspace/


## Using

Office Documents, like DOCX, are including a new action in ADW ("Edit in Word online"). After a user clicks this option in ADW, the user is autenticated in Azure Active Directory, the document is copied to "office.com" space and a new tab is opened to start editing the document. Once the document has been edited, close the tab and click "End Editing" in ADW to create a new version with the changes made in "office.com".