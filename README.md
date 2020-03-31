# How to control access to Hyperledger Sawtooth's REST API

*Keywords: Linux, Ubuntu, Docker, Docker-Compose, Apache, Reverse Proxy, HTTP, HTTPS, SSL, Uncomplicated Firewall, Hyperledger Sawtooth, REST API.*

### Overview

Using a reverse proxy Hyperledger Sawtooth can support authorization and allow cross-origin access (CORS) to the REST API. 

The role of firewall is to filter and limit access to Hyperledger Sawtooth's REST API commands.

Each Docker network is associated with a bridge interface on the Ubuntu host, and Linux firewall rules are defined to filter traffic between these interfaces.

By default, the Docker daemon listens for connections on a socket to accept requests sent from Hyperledger Sawtooths' clients to [Hyperledger Sawtooth's REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/rest_api/endpoint_specs.html) endpoint.

The reverse proxy server takes requests from a Sawtooth client and forwards these requests to Hyperledger Sawtooth's REST API server. 

The following are steps how to install, configure, and run Apache Reverse Proxy and Linux Firewall to control access to Hyperledger Sawtooth's REST API.

### Prerequisites

The configuration uses [Docker Engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/) to run Hyperledger Sawtooth's REST API Server on Ubuntu.

Read procedures how to set up [Hyperledger Sawtooth using Docker](https://sawtooth.hyperledger.org/docs/core/releases/latest/app_developers_guide/docker.html) for a Sawtooth Node.

### Step-by-Step

Step 1. Docker Engine and Docker Compose

Install the latest versions of Docker Engine and Docker Compose on Ubuntu.

Step 2. Docker containers

Build Docker image and start Docker container for the Hyperledger Sawtooth's REST API.

Set up Sawtooth node to fetch information from the blockchain. The Sawtooth client communicates with Sawtooth node by REST API requests, including signed transactions and batches.

Download or clone Hyperledger Sawtooth's blockchain project for [Supply Chain](https://github.com/hyperledger/sawtooth-supply-chain). The Docker Compose's yaml file contains details about services, networks, and volumes for setting up the Hyperledger Sawtooth's REST API.

The Supply Chain application runs on top of Hyperledger Sawtooth. Follow the Supply Chain instructions to create Docker containers for Hyperledger Sawtooth's REST API. From the project directory, start up Docker containers by running docker-compose up command.

`$ docker start supply-rest-api`

Hyperledger Sawtooth's blockchain REST API will be available at http://localhost:8024

Step 3. Reverse Proxy

The reverse proxy provides a single point of authentication and handles incoming HTTPS connections, decrypting the requests and passing unencrypted HTTP requests on to Hyperledger Sawtooth's REST API.

Install [Apache](https://httpd.apache.org/docs/2.4/) and enable the required modules. 

```
$ sudo apt-get update

$ sudo apt-get install -y apache2

$ sudo a2enmod ssl

$ sudo a2enmod headers

$ sudo a2enmod proxy_http

$ sudo systemctl restart apache2
```

Create a password file for the user of sawtooth and repeat htpasswd command to generate passwords for other users, without the -c option. 

`$ sudo htpasswd -c /etc/apache2/.htpassword sawtooth`

Generate SSL certificate, public and private keys to make requests to Hyperledger Sawtooth's REST API.

The openssl command generates an error if .rnd file does not already exist in the user's directory.

```
$ touch ~/.rnd
$ sudo mkdir /etc/apache2/keys
$ sudo openssl req -x509 -nodes -days 7300 -newkey rsa:2048 \
  -subj /C=US/ST=MN/L=Mpls/O=Sawtooth/CN=sawtooth \
  -keyout /tmp/.ssl.key \
  -out /tmp/.ssl.crt
```

Copy SSL certificate and keys to /etc/apache2/keys directory.

```
$ cp /tmp/.ssl.crt /etc/apache2/keys/.

$ cp /tmp/.ssl.key /etc/apache2/keys/.
```

Create an Apache configuration file for Reverse Proxy.

`$ sudo vi /etc/apache2/sites-available/000-default.conf`

ProxyPassReverse directive is required to ensure that headers are modified to point to the reverse proxy instead of Hyperledger Sawtooth's REST API.

Add the following contents to 000-default.conf file.

	<VirtualHost *:443>
	    ServerName sawtooth
	    ServerAdmin sawtooth@sawtooth
	    DocumentRoot /var/www/html

	    SSLEngine on
	    SSLCertificateFile /etc/apache2/keys/.ssl.crt
	    SSLCertificateKeyFile /etc/apache2/keys/.ssl.key
	    RequestHeader set X-Forwarded-Proto "https"

	    <Location /sawtooth>
		Options Indexes FollowSymLinks
		AllowOverride None
		AuthType Basic
		AuthName "Enter password"
		AuthUserFile "/etc/apache2/.htpassword"
		Require user sawtooth
		Require all denied
		ProxyPass http://localhost:8008
		ProxyPassReverse http://localhost:8008
		RequestHeader set X-Forwarded-Path "/sawtooth"
	    </Location>
	</VirtualHost>`

Hyperledger Sawtooth's REST API builds a link using types of “X-Forwarded” headers X-Forwarded-Host, X-Forwarded-Proto and X-Forwarded-Path for HTTP requests. X-Forwarded-Path is necessary if the reverse proxy endpoints do not map directly to the REST API endpoints.

Query the reverse proxy with SSL authorization by curl command or browser's location bar.

`https://localhost/sawtooth/blocks`
  
Step 4. Filtering Packets

Access control refers to way of controlling access to Hyperledger Sawtooth's REST API by firewall.

### References

[Using a Proxy Server to Authorize the REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/sysadmin_guide/rest_auth_proxy.html)
