# OpenWRT NAS (FileRun + Cloudflared)

NAS (Network Attached Storage) is a device that functions to store and share data across multiple computers, which can be accessed remotely.

This project using FileRun + Cloudflare
- [FileRun - Share files with your clients and colleagues](https://filerun.com/ "FileRun - Share files with your clients and colleagues")
- [Tunnel using Cloudflared (Cloudflare Zero Trust)](https://www.cloudflare.com/products/zero-trust/ "Tunnel using Cloudflared (Cloudflare Zero Trust)")

#### Pre-Requisites

- OpenWRT v18.06+
- Docker
- Internet Connection and some coffe + snacks

#### Docker
*(skip this step if you have already install docker on your openwrt)*

Login to OpenWRT SSH & Install with docker with two command below :
> opkg update
opkg install docker docker-compose dockerd luci-app-dockerman

#### FileRun
For ARM32 (OpenWRT) please use [this guide.](https://docs.filerun.com/docker-arm "this guide.")
For ARM64 (OpenWRT) please use [this guide.](https://docs.filerun.com/docker-arm64 "this guide.")

Review `docker-compose.yml` configuration file. below example of arm64 configuration files.

```
version: '2'

services:
  db:
    image: mariadb:10.1
    environment:
      MYSQL_ROOT_PASSWORD: your_mysql_root_password
      MYSQL_USER: your_filerun_username
      MYSQL_PASSWORD: your_filerun_password
      MYSQL_DATABASE: your_filerun_database
    volumes:
      - /filerun/db:/var/lib/mysql

  web:
    image: filerun/filerun
    environment:
      FR_DB_HOST: db
      FR_DB_PORT: 3306
      FR_DB_NAME: your_filerun_database
      FR_DB_USER: your_filerun_username
      FR_DB_PASS: your_filerun_password
      APACHE_RUN_USER: www-data
      APACHE_RUN_USER_ID: 33
      APACHE_RUN_GROUP: www-data
      APACHE_RUN_GROUP_ID: 33
    depends_on:
      - db
    links:
      - db:db
    ports:
      - "8181:80"
    volumes:
      - /filerun/html:/var/www/html
      - /this-is-path-for-store-files-to-your-storage:/user-files
```
For line 31 : The filerun will running on 8181 port. you can customize the port but don't use 443, 80, or already used ports.

For line 61 : before `:/user-files` -> (`/this-is-path-for-store-files-to-your-storage`). 
that is your local storage for store all uploaded files to FileRun App. don't remove the `:/user-files` that is a docker path.

And start FileRun up using the following command:
> docker-compose up -d

FileRun should be now up and running and you can access it with your browser.

#### Tunnel using Cloudflare
See latest version of cloudflared at [https://github.com/cloudflare/cloudflared/releases](https://github.com/cloudflare/cloudflared/releases "https://github.com/cloudflare/cloudflared/releases")

Please note the versions and customize version variable below, change the arch to `arm` if you using ARM32 OpenWRT

Then SSH as root into the device to download and install cloudflared binary :

```VERSION="2022.12.1"
ARCH="arm64"
curl -O -L \
https://github.com/cloudflare/cloudflared/releases/download/${VERSION}/cloudflared-linux-${ARCH} \
&& chmod +x cloudflared-linux-${ARCH} \
&& mv cloudflared-linux-${ARCH} /usr/bin/cloudflared```

