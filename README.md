# How to control access to Hyperledger Sawtooth's REST API

*Keywords: Linux, Ubuntu, Docker, Docker-Compose, Apache, Nginx, Reverse Proxy, HTTP, HTTPS, SSL, Uncomplicated Firewall, Hyperledger Sawtooth, REST API.*

### Overview

Using a Reverse Proxy Hyperledger Sawtooth supports Authorization and allows Cross-Origin Resource Sharing (CORS) to access from a different domain to Hyperledger Sawtooth's REST API.

The role of firewall is to filter and limit access to Hyperledger Sawtooth's REST API.

Docker is an easy way to get Hyperledger Sawtooth up and running REST API. Instead of forwarding the requests directly to the Docker container, the Reverse Proxy is listening for incoming HTTPS requests and forward them to Hyperledger Sawtooth's REST API.

Each Docker network is associated with a bridge interface on the Ubuntu host and Linux Firewall rules are defined to filter or deny traffic between these interfaces.

By default, the Docker daemon listens for connections on a socket to accept HTTP requests sent from Hyperledger Sawtooths' clients to [Hyperledger Sawtooth's REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/rest_api/endpoint_specs.html) endpoint.

The Reverse Proxy receives HTTP requests from a Sawtooth client and forwards HTTP requests to Hyperledger Sawtooth's REST API. 

The following are major steps how to install, configure, and run either Apache or Nginx as Reverse Proxy and Linux Firewall to control access to Hyperledger Sawtooth's REST API.

### Prerequisites

The environment uses [Docker Engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/) to run Hyperledger Sawtooth's REST API Server on Ubuntu.

Read procedures to know how to set up [Hyperledger Sawtooth using Docker](https://sawtooth.hyperledger.org/docs/core/releases/latest/app_developers_guide/docker.html) for a Sawtooth Node.

### Step-by-Step

*Step 1. Docker Engine and Docker Compose*

Install the latest versions of Docker Engine and Docker Compose on Ubuntu.

*Step 2. Docker Containers*

Build Docker image and start Docker container for the Hyperledger Sawtooth's REST API.

Set up Sawtooth node to fetch information from the blockchain. The Sawtooth client communicates with Sawtooth node by REST API requests, including signed transactions and batches.

Download or clone Hyperledger Sawtooth's blockchain project for [Supply Chain](https://github.com/hyperledger/sawtooth-supply-chain). The Docker Compose's yaml file contains details about services, networks, and volumes for setting up the Hyperledger Sawtooth's REST API.

The project is an example blockchain application for Supply Chain using Hyperledger Sawtooth's REST API. Follow the Supply Chain instructions to create Docker containers for Hyperledger Sawtooth's REST API. From the project directory, start up Docker containers by running docker-compose up command.

`$ docker start supply-rest-api`

Hyperledger Sawtooth's blockchain REST API will be available at http://localhost:8024

*Step 3. Reverse Proxy*

The Reverse Proxy provides a single point of Authentication and it handles incoming HTTPS connections, decrypting the requests and passing unencrypted HTTP requests on to Hyperledger Sawtooth's REST API.

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

Hyperledger Sawtooth's REST API builds a link using types of “X-Forwarded” headers X-Forwarded-Host, X-Forwarded-Proto and X-Forwarded-Path for HTTP requests. To put X-Forwarded-Path is necessary if the Reverse Proxy endpoints do not map directly to the REST API endpoints.

Query the reverse proxy with SSL authorization by curl command or browser's location bar.

`https://localhost/sawtooth/blocks`

Apache listens on the ports determined in /etc/apache2/ports.conf file.

*Step 4. The rules of iptables*

Access control is a way to control access to Hyperledger Sawtooth's REST API. 

The iptables is an userspace interface to manage packet filtering and port filtering. 

Uncomplicated firewall (ufw) is way to add or remove rules to the [iptables](https://docs.docker.com/network/iptables/).

Docker container allows the public network to access the services provided by the Docker. 

To be able to reach Docker container from the public network, you have to publish its [port(s)](https://docs.docker.com/config/containers/container-networking/) to the host network. 

To restrict connections to the Docker host, insert iptables rules at the top of the DOCKER-USER chain. 

Rules can be added by denoting the port number to allow or to deny traffic on the port. 

The rules are processed in the order of the file top to bottom. 

Edit /etc/ufw/after.rule file to alter default rules for the iptables.

Configure /etc/ufw/after.rule file to allow the packets to traverse through DOCKER-USER chain by RETURN rule or restrict incoming packets from a public network by DROP rule.

### Optional - Nginx

The configuration file default.conf in Nginx uses proxy_pass directive to pass requests to the location /sawtooth/blocks to Docker container's URL.

By default, Docker reads container’s file /etc/resolv.conf to use a DNS and Nginx's configuration file needs to include the resolver directive to explicitly specify the DNS to resolve hostnames.

```server {
	listen 443 ssl http2;
	server_name reverse_proxy;
	ssl_certificate /etc/ssl/certs/nginx/site.crt;
	ssl_certificate_key /etc/ssl/certs/nginx/site.key;
	include /etc/nginx/includes/ssl.conf;
	resolver 127.0.0.11 valid=30s;
	location /sawtooth/blocks {
		include /etc/nginx/includes/proxy.conf;
		set $target supply-rest-api:8008/blocks;
		proxy_pass http://$target;
	}
	access_log off;
	error_log  /var/log/nginx/error.log error;
}```


### References

[Using a Proxy Server to Authorize the REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/sysadmin_guide/rest_auth_proxy.html)
