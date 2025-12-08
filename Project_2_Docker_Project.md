## Installation of Docker
Go to docs.docker.com/engine/install/. We will be following this guide to install Docker.
Choose Debian as we have a Kali Linux distro
Run the following command to uninstall old versions
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
``` 
Next, we must setup the Docker `apt` repository before installing the Docker Engine.
Run the following commands to set up Docker's `apt` repository
```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker/sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: bookworm
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt-get update
```
Note: In the `Suites:` section, as we are in a Kali Linux distribution, a derivative distribution of Debian, replace this part as shown with `bookworm` instead of the `$(. /etc/os-release && echo "$VERSION_CODENAME")`shown in the guide.

Install Docker packages:
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
After this is finished, we can verify that Docker is running by using:
```bash
sudo systemctl status docker
```
Although, some systems may have the automatic start after installation behavior disabled and requires a manual start:
```bash
sudo systemctl start docker
```
Afterwards, verify the installation is successful by running the `hello-world` image:
```bash
sudo docker run hello-world
```
## Docker Compose Application
For the choice of application, I chose Snipe-IT, a web-based management system used to track and manage organizational assets.

To begin, create a project directory for Snipe-IT and then a .yml file to paste the Snipe-IT image into
```bash
# Create and change directory into snipeit-docker
mkdir snipeit-docker
cd snipeit-docker

# Create .yml file
nano docker-compose.yml
```
After creating the YAML file, paste the Snipe-IT image into the file
```bash
# Compose file for production.

volumes:
  db_data:
  storage:

services:
  app:
    image: snipe/snipe-it:${APP_VERSION:-latest}
    restart: unless-stopped
    volumes:
      - storage:/var/lib/snipeit
    ports:
      - "${APP_PORT:-8000}:80"
    depends_on:
      db:
        condition: service_healthy
        restart: true
    env_file:
      - .env

  db:
    image: mariadb:11.4.7
    restart: unless-stopped
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    healthcheck:
      # https://mariadb.com/kb/en/using-healthcheck-sh/#compose-file-example
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 5s
      timeout: 1s
      retries: 5
```
After, `Ctrl + O` and `Ctrl + X` to write the changes and exit out the file.
Next, create the `.env` file to paste the environment variables into the file
```bash
# Create .env file
nano .env

# Paste .env contents into file
# --------------------------------------------
# REQUIRED: DOCKER SPECIFIC SETTINGS
# --------------------------------------------
APP_VERSION=
APP_PORT=8000

# --------------------------------------------
# REQUIRED: BASIC APP SETTINGS
# --------------------------------------------
APP_ENV=production
APP_DEBUG=false
# Please regenerate the APP_KEY value by calling `docker compose run --rm app php artisan key:generate --show`. Copy paste the value here
APP_KEY=base64:3ilviXqB9u6DX1NRcyWGJ+sjySF+H18CPDGb3+IVwMQ=
APP_URL=http://localhost:8000
# https://en.wikipedia.org/wiki/List_of_tz_database_time_zones - TZ identifier
APP_TIMEZONE='UTC'
APP_LOCALE=en-US
MAX_RESULTS=500

# --------------------------------------------
# REQUIRED: UPLOADED FILE STORAGE SETTINGS
# --------------------------------------------
PRIVATE_FILESYSTEM_DISK=local
PUBLIC_FILESYSTEM_DISK=local_public

# --------------------------------------------
# REQUIRED: DATABASE SETTINGS
# --------------------------------------------
DB_CONNECTION=mysql
DB_HOST=db
DB_SOCKET=null
DB_PORT='3306'
DB_DATABASE=snipeit
DB_USERNAME=snipeit
DB_PASSWORD=changeme1234
MYSQL_ROOT_PASSWORD=changeme1234
DB_PREFIX=null
DB_DUMP_PATH='/usr/bin'
DB_DUMP_SKIP_SSL=true
DB_CHARSET=utf8mb4
DB_COLLATION=utf8mb4_unicode_ci

# --------------------------------------------
# OPTIONAL: SSL DATABASE SETTINGS
# --------------------------------------------
DB_SSL=false
DB_SSL_IS_PAAS=false
DB_SSL_KEY_PATH=null
DB_SSL_CERT_PATH=null
DB_SSL_CA_PATH=null
DB_SSL_CIPHER=null
DB_SSL_VERIFY_SERVER=null

# --------------------------------------------
# REQUIRED: OUTGOING MAIL SERVER SETTINGS
# --------------------------------------------
MAIL_MAILER=smtp
MAIL_HOST=mailhog
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_TLS_VERIFY_PEER=true
MAIL_FROM_ADDR=you@example.com
MAIL_FROM_NAME='Snipe-IT'
MAIL_REPLYTO_ADDR=you@example.com
MAIL_REPLYTO_NAME='Snipe-IT'
MAIL_AUTO_EMBED_METHOD='attachment'

# --------------------------------------------
# REQUIRED: DATA PROTECTION
# --------------------------------------------
ALLOW_BACKUP_DELETE=false
ALLOW_DATA_PURGE=false

# --------------------------------------------
# REQUIRED: IMAGE LIBRARY
# This should be gd or imagick
# --------------------------------------------
IMAGE_LIB=gd

# --------------------------------------------
# OPTIONAL: BACKUP SETTINGS
# --------------------------------------------
MAIL_BACKUP_NOTIFICATION_DRIVER=null
MAIL_BACKUP_NOTIFICATION_ADDRESS=null
BACKUP_ENV=true

# --------------------------------------------
# OPTIONAL: CHANGE PHP UPLOAD LIMITS (UNCOMMENT WHEN NEEDING TO BE CHANGED)
# --------------------------------------------
#PHP_UPLOAD_LIMIT=10
#PHP_POST_MAX_SIZE=10
#PHP_UPLOAD_MAX_FILESIZE=10
#PHP_MEMORY_LIMIT=10


# --------------------------------------------
# OPTIONAL: SESSION SETTINGS
# --------------------------------------------
SESSION_LIFETIME=12000
EXPIRE_ON_CLOSE=false
ENCRYPT=false
COOKIE_NAME=snipeit_session
COOKIE_DOMAIN=null
SECURE_COOKIES=false
API_TOKEN_EXPIRATION_YEARS=40

# --------------------------------------------
# OPTIONAL: SECURITY HEADER SETTINGS
# --------------------------------------------
APP_TRUSTED_PROXIES=192.168.1.1,10.0.0.1,172.16.0.0/12
ALLOW_IFRAMING=false
REFERRER_POLICY=same-origin
ENABLE_CSP=false
CORS_ALLOWED_ORIGINS=null
ENABLE_HSTS=false

# --------------------------------------------
# OPTIONAL: CACHE SETTINGS
# --------------------------------------------
CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_DRIVER=sync
CACHE_PREFIX=snipeit

# --------------------------------------------
# OPTIONAL: REDIS SETTINGS
# --------------------------------------------
REDIS_HOST=null
REDIS_PASSWORD=null
REDIS_PORT=6379

# --------------------------------------------
# OPTIONAL: MEMCACHED SETTINGS
# --------------------------------------------
MEMCACHED_HOST=null
MEMCACHED_PORT=null

# --------------------------------------------
# OPTIONAL: PUBLIC S3 Settings
# --------------------------------------------
PUBLIC_AWS_SECRET_ACCESS_KEY=null
PUBLIC_AWS_ACCESS_KEY_ID=null
PUBLIC_AWS_DEFAULT_REGION=null
PUBLIC_AWS_BUCKET=null
PUBLIC_AWS_URL=null
PUBLIC_AWS_BUCKET_ROOT=null

# --------------------------------------------
# OPTIONAL: PRIVATE S3 Settings
# --------------------------------------------
PRIVATE_AWS_ACCESS_KEY_ID=null
PRIVATE_AWS_SECRET_ACCESS_KEY=null
PRIVATE_AWS_DEFAULT_REGION=null
PRIVATE_AWS_BUCKET=null
PRIVATE_AWS_URL=null
PRIVATE_AWS_BUCKET_ROOT=null

# --------------------------------------------
# OPTIONAL: AWS Settings
# --------------------------------------------
AWS_ACCESS_KEY_ID=null
AWS_SECRET_ACCESS_KEY=null
AWS_DEFAULT_REGION=null

# --------------------------------------------
# OPTIONAL: LOGIN THROTTLING
# --------------------------------------------
LOGIN_MAX_ATTEMPTS=5
LOGIN_LOCKOUT_DURATION=60
RESET_PASSWORD_LINK_EXPIRES=900
INVITE_PASSWORD_LINK_EXPIRES=1500

# --------------------------------------------
# OPTIONAL: MISC
# --------------------------------------------
LOG_CHANNEL=stderr
LOG_MAX_DAYS=10
APP_LOCKED=false
APP_CIPHER=AES-256-CBC
APP_FORCE_TLS=false
GOOGLE_MAPS_API=
LDAP_MEM_LIM=500M
LDAP_TIME_LIM=600
```
Edit the following lines to set a strong password for the Snipe-IT user and root user and then set the access to the application to your VM with your IP address
```bash
DB_PASSWORD:snipeit3353
MYSQL_ROOT_PASSWORD:admin3353
APP_URL:http://10.0.2.15:8000
```
The Snipe-IT web app is now ready to run:
```bash
sudo docker compose up -d
```
Using the same `APP_URL` go to a browser and visit the application
```bash
http://10.0.2.15:8000
```
Images of being on the application:
![[Pasted image 20251114113936.png]]
![[Pasted image 20251114114004.png]]
