# How to control access to Hyperledger Sawtooth's REST API

*Keywords: Linux, Ubuntu, Docker, Docker-Compose, Apache, Reverse Proxy, HTTP, HTTPS, SSL, Uncomplicated Firewall, Hyperledger Sawtooth, REST API.*

### Overview

Hyperledger Sawtooth's REST API does not support authorization, but it is possible to control using a reverse proxy and firewall the requests sent to REST API. 

Each Docker network is associated with a bridge interface on the Ubuntu host, and Linux firewall rules are defined to filter traffic between these interfaces.

By default, the Docker daemon listens for connections on a socket to accept requests sent from Hyperledger Sawtooths' clients to [Hyperledger Sawtooth's REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/rest_api/endpoint_specs.html) endpoint.

The following are steps how to install, configure, and run Apache Reverse Proxy and Linux Firewall to control access to Hyperledger Sawtooth's REST API.

### Prerequisites

The configuration uses [Docker Engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/) and Docker Compose to run Hyperledger Sawtooth's REST API Server on Ubuntu.

Read procedures how to set up [Hyperledger Sawtooth using Docker](https://sawtooth.hyperledger.org/docs/core/releases/latest/app_developers_guide/docker.html) for a Sawtooth Node.


### References

[Using a Proxy Server to Authorize the REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/sysadmin_guide/rest_auth_proxy.html)
