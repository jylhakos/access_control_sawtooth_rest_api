# How to control access to Hyperledger Sawtooth's REST API

*Keywords: Linux, Ubuntu, Docker, Docker-Compose, Apache, Reverse Proxy, HTTP, HTTPS, SSL, Uncomplicated Firewall, Hyperledger Sawtooth, REST API.*

### Overview

Hyperledger Sawtooth's REST API does not support authorization, but it is possible to control using a reverse proxy and firewall the requests sent to REST API. 

Each Docker network is associated with a bridge interface on the Ubuntu host, and Linux firewall rules are defined to filter traffic between these interfaces.

By default, the Docker daemon listens for connections on a socket to accept requests sent from Hyperledger Sawtooths' clients to [Hyperledger Sawtooth's REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/rest_api/endpoint_specs.html) endpoint.

The reverse proxy server takes requests from a Sawtooth client and forwards these requests to Hyperledger Sawtooth's REST API server. 

The following are steps how to install, configure, and run Apache Reverse Proxy and Linux Firewall to control access to Hyperledger Sawtooth's REST API.

### Prerequisites

The configuration uses [Docker Engine](https://docs.docker.com/install/linux/docker-ce/ubuntu/) and [Docker Compose](https://docs.docker.com/compose/install/) to run Hyperledger Sawtooth's REST API Server on Ubuntu.

Read procedures how to set up [Hyperledger Sawtooth using Docker](https://sawtooth.hyperledger.org/docs/core/releases/latest/app_developers_guide/docker.html) for a Sawtooth Node.

### Step-by-Step

Step 1. Install the latest versions of Docker Engine and Docker Compose on Ubuntu.

Step 2: Build Docker image and start Docker container for the Hyperledger Sawtooth's REST API server.

Set up Sawtooth node to fetch information from the blockchain. The Sawtooth client communicates with Sawtooth node by REST API requests, including signed transactions and batches.

Download or clone Hyperledger Sawtooth's blockchain project for [Supply Chain](https://github.com/hyperledger/sawtooth-supply-chain).

Follow the Supply Chain instructions to run supply-rest-api server on Ubuntu.

`$ sudo -u sawtooth sawtooth-rest-api -vvv`

Hyperledger Sawtooth's blockchain REST API will be available at http://localhost:8024

### References

[Using a Proxy Server to Authorize the REST API](https://sawtooth.hyperledger.org/docs/core/releases/latest/sysadmin_guide/rest_auth_proxy.html)
