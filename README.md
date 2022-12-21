# OpenWRT NAS (FileRun + Cloudflared)

NAS (Network Attached Storage) is a device that functions to store and share data across multiple computers, which can be accessed remotely.

This project using FileRun + Cloudflare
- [FileRun - Share files with your clients and colleagues](https://filerun.com/ "FileRun - Share files with your clients and colleagues")
- [Tunnel using Cloudflared (Cloudflare Zero Trust)](https://www.cloudflare.com/products/zero-trust/ "Tunnel using Cloudflared (Cloudflare Zero Trust)")

## Pre-Requisites

- OpenWRT v18.06+
- Docker
- Internet Connection and some coffe + snacks
- AND READ F*CK*NG CAREFULLY THIS DOCS BEFORE OPEN THE ISSUE

## Docker
*(skip this step if you have already install docker on your openwrt)*

Login to OpenWRT SSH & Install with docker with two command below :

```sh
$ opkg update
$ opkg install docker docker-compose dockerd luci-app-dockerman
```

## FileRun
- For ARM32 (OpenWRT) please use [this guide.](https://docs.filerun.com/docker-arm "this guide.")
- For ARM64 (OpenWRT) please use [this guide.](https://docs.filerun.com/docker-arm64 "this guide.")

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
For The Port : The FileRun App will running on 8181 port. you can customize the port but don't use 443, 80, or already used ports.

LAST LINE : Before `:/user-files` -> (`/this-is-path-for-store-files-to-your-storage`). 
that is your local storage for store all uploaded files to FileRun App. don't remove the `:/user-files` that is a docker path.

And start FileRun up using the following command:
```sh
$ docker-compose up -d
```

FileRun should be now up and running and you can access it with your browser.

## Tunnel using Cloudflare
### Pre-Requisites
- Have website domain in your cloudflare dns for creating subdomain

### 1. Download and install cloudflared

See latest version of cloudflared at [https://github.com/cloudflare/cloudflared/releases](https://github.com/cloudflare/cloudflared/releases "https://github.com/cloudflare/cloudflared/releases").

**Take note of the latest release version**

Then Login to SSH as root into the OpenWRT for starting setup the cloudflared :
```
VERSION="2022.12.1"
ARCH="arm64"

curl -O -L \
https://github.com/cloudflare/cloudflared/releases/download/${VERSION}/cloudflared-linux-${ARCH} \
&& chmod +x cloudflared-linux-${ARCH} \
&& mv cloudflared-linux-${ARCH} /usr/bin/cloudflared
```
Please note the versions and customize version variable above, change the arch to `arm` if you using ARM32 OpenWRT

Now you can run cloudflared command.

### 2. Authenticate cloudflared

```sh
$ cloudflared tunnel login
```
Running this command will:

- Open a browser window and prompt you to log in to your Cloudflare account. After logging in to your account, select your hostname.
- Generate an account certificate, the [cert.pem file](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#cert-pem), in the [default `cloudflared` directory](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#default-cloudflared-directory).

**Download the generated account certificate file and copy to `/usr/local/etc/cloudflared` OpenWRT Directory**

### 3. Create a tunnel and give it a name

```sh
$ cloudflared tunnel create <NAME>

Example
$ cloudflared tunnel create mytunnel
```

Running this command will:

- Create a tunnel by establishing a persistent relationship between the [name you provide](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#tunnel-name) and a [UUID](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#tunnel-uuid) for your tunnel. At this point, no connection is active within the tunnel yet.
- Generate a [tunnel credentials file](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#credentials-file) in the [default `cloudflared` directory](/cloudflare-one/connections/connect-apps/install-and-setup/tunnel-useful-terms/#default-cloudflared-directory).

**Take note of the tunnel’s UUID and the path to your tunnel’s credentials file.**

### 4. Create a configuration file

Create a `config.yml` file into `/usr/local/etc/cloudflared` directory
You can use `nano` or `vim`
```sh
$ vim /usr/local/etc/cloudflared/config.yml
```

Add the following fields to the file:

```txt
url: http://localhost:8181
tunnel: <Tunnel-UUID>
credentials-file: /path-to-credentials-file/<Tunnel-UUID>.json
```
Example :
```txt
url: http://localhost:8181
tunnel: 7f6812c3-7bcb-4bc8-21c7-5315cbd56e4e
credentials-file: /usr/local/etc/cloudflared/7f6812c3-7bcb-4bc8-21c7-5315cbd56e4e.json
```

- **Customize the url port to your FileRun app**
- **Please ReCheck The Tunnel UUID on Cloudflare Zero Thrust Dashboard**

### 5. Create routing traffic

Now assign a CNAME record that points traffic to your tunnel subdomain.

```sh
$ cloudflared tunnel route dns <TUNNEL-UUID> <SUB-DOMAIN>

Example
$ cloudflared tunnel route dns 7f6812c3-7bcb-4bc8-21c7-5315cbd56e4e cloud.myprivatenas.com
```
Running this command will:
- Create CNAME Record on your DNS Website on Cloudflare

### 6. Run the tunnel

Run the tunnel to proxy incoming traffic from the tunnel to any number of services running locally on your origin.

```sh
$ cloudflared tunnel run <TUNNEL-UUID>

Example
$ cloudflared tunnel run 7f6812c3-7bcb-4bc8-21c7-5315cbd56e4e
```
Done!
Check your subdomain to make sure the tunneling running!

**to make your tunnel running automatically on system boot, please follow step below**

## Run Cloudflared Tunnel as a service on OpenWRT
