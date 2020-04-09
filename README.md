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

Install apache2-utils package with the following command.

`$ sudo apt-get install apache2-utils`

The openssl command generates an error if .rnd file does not already exist in the user's directory.

```
$ touch ~/.rnd
$ sudo mkdir /etc/apache2/keys
$ sudo openssl req -x509 -nodes -days 7300 -newkey rsa:2048 \
  -subj /C=US/ST=MN/L=Mpls/O=Sawtooth/CN=sawtooth \
  -keyout /tmp/.ssl.key \
  -out /tmp/.ssl.crt
```
Secure Socket Layer (SSL) is used to encrypt network transactions. 

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

### Apache deployed on Docker (Optional)

Apache httpd image can be deployed on Docker Container to authenticate blockchain users connected to Sawtooth's REST API services.

Configure host [volumes](https://docs.docker.com/storage/volumes/) to map SSL certificate and private keys to /usr/local/apache2/conf/ in the docker-compose.yaml file.

```
version: '3.0'

volumes:
  config:

networks:
  supply-chain-network:
    external:
      name: supply-chain-network

services:

  rest-api:
      image: hyperledger/sawtooth-rest-api:1.0
      container_name: supply-rest-api
      hostname: rest-api
      expose:
        - 8008
      depends_on:
        - validator
      entrypoint: |
        sawtooth-rest-api -vv
        --connect tcp://validator:4004
        --bind rest-api:8008
      networks:
        - supply-chain-network

  apache-proxy:
    image: "httpd:2.4"
    container_name: apache-proxy
    hostname: apache-proxy
    volumes:
      - "./config:/usr/local/apache2/conf/"
    ports:
    - 443:443
    networks:
      - supply-chain-network
```

Configure Apache and Virtual Hosts in /usr/local/apache2/conf/httpd.conf file.

```
<VirtualHost *:443>
	ServerName localhost:443
	ServerAdmin demo@DEMO
	DocumentRoot /var/www/html
	SSLEngine on
	SSLCertificateFile /usr/local/apache2/conf/ssl.crt
	SSLCertificateKeyFile /usr/local/apache2/conf/ssl.key
	RequestHeader set X-Forwarded-Proto	"https"
	ProxyPass /error/ !
	<Location /sawtooth/blocks>
		Options Indexes FollowSymLinks
		AllowOverride None
		AuthType Basic
		AuthName "Enter password"
        AuthUserFile "/usr/local/apache2/conf/htpassword"
		Require user sawtooth
		Require all denied
		ProxyPass http://rest-api:8008/blocks
		ProxyPassReverse http://rest-api:8008/blocks
		RequestHeader set X-Forwarded-Path "/sawtooth/blocks"
	</Location>
</VirtualHost>
```

Configure a hostname for the service to connect dependent Docker containers on Docker network.

If Docker version is earlier than 3.0, then Docker container's IP address is associated to hosts file /etc/hosts after running Docker container.

Docker inspect provides detailed information on IP addresses of all Docker Containers.

` $ docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)`

Docker downloads Apache httpd image as part of the build. 

Start Docker container by [Docker Compose](https://docs.docker.com/compose/) command.

`$ docker-compose up -d`

### Dockerized Nginx (Optional)

Nginx is an open source Reverse Proxy server for HTTP and HTTPS protocols.

To pass HTTPS requests to Hyperledger Sawtooth's REST API, the [proxy_pass directive](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) is specified inside a location. 

Configure default.conf file to specify proxy_pass directive with Docker container's name in the location /sawtooth/blocks block.

To [configure Nginx to use HTTPS](https://nginx.org/en/docs/http/configuring_https_servers.html) protocol, the SSL parameter must be enabled on listening sockets in the server block, and the locations of the SSL certificate and private key files need to be specified in default.conf file.

Nginx can [restrict access](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/) to the location assigned with proxy_pass directive by implementing Authentication.

Generate a signed certificate using apache2-utils command.

`$ sudo htpasswd -c etc/nginx/.htpasswd sawtooth`

Specify in the location block auth_basic directive for authentication and auth_basic_user_file directive with a path to the password file.

By default, Docker reads container’s file /etc/resolv.conf to use a DNS, therefore Nginx's configuration file includes the [resolver directive](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#resolver) to explicitly specify the DNS to resolve hostnames.

```
server {
	listen 443 ssl http2; 
	server_name reverse_proxy;
	ssl_certificate /etc/ssl/certs/nginx/site.crt;
	ssl_certificate_key /etc/ssl/certs/nginx/site.key;
	include /etc/nginx/includes/ssl.conf;
	resolver 127.0.0.11 valid=30s;
	location /sawtooth/blocks {
		include /etc/nginx/includes/proxy.conf;
		auth_basic "Restricted Content";
        	auth_basic_user_file /etc/nginx/htpasswd;
		set $target supply-rest-api:8008/blocks;
		proxy_pass http://$target;
	}
	access_log off;
	error_log  /var/log/nginx/error.log error;
}
```

Docker Compose uses a YAML file to configure Nginx’s services. 

```
version: '2.1'

networks:
  supply-chain-network:
    external:
      name: supply-chain-network

services:
  rest-api:
    image: hyperledger/sawtooth-rest-api:1.0
    container_name: supply-rest-api
    expose:
      - 8008
    depends_on:
      - validator
    entrypoint: |
      sawtooth-rest-api -vv
        --connect tcp://validator:4004
        --bind rest-api:8008
    networks:
      - supply-chain-network

  nginx-proxy:
    image: nginx-proxy
    container_name: nginx-proxy
    networks:
      - supply-chain-network
    ports:
      - 443:443
```

The [docker-compose](https://docs.docker.com/compose/reference/up/) up command uses Dockerfile to build Docker container with Nginx image.

`$ docker-compose up --build` 

### References

[Using a Proxy Server to Authorize the REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/sysadmin_guide/rest_auth_proxy.html)
