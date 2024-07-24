# Passenger Quickstart Guide

Buckle up for a smooth and safe remote access.

## Get Started

15 min.

In this unit you will learn how to install Passenger using Docker.

At the end of this section, you will have a working Passenger instance behind a reverse proxy. The reverse proxy used in this guide is traefik, but any other preferable option would work fine.

## Prerequisites

Before you begin, make sure you have the following tools installed:

* [Docker](https://docs.docker.com/engine/install/) - Containerization platform for running Passenger.
* [Docker Compose](https://docs.docker.com/compose/install/) - Used to manage multi-container Docker applications (only necessary for older Docker versions; Compose V2 is included in newer versions).
* [KeePass](https://keepass.info/) - Used for the local and static vault

## Installation

This step-by-step installation guide is intended for testing and development purposes and should not be used in a production environment.

### Step 1: Prepare

Open a terminal session and create a folder `/opt/passenger` (or any other location). Create following folders:

```bash
cd /opt/passenger
mkdir config
mkdir db
```

Run `vi .env` and paste following content:

```bash
WRITE_FQDN=write.example.com
READ_FQDN=read.example.com
POSTMASTER_EMAIL=your_email@example.com
EXPOSED_SSH_PORT=22
```

> **_NOTE:_**
> * **DNS Entries**: Make sure that you have configured DNS entries for `write.example.com` and `read.example.com` to point to the machine where you are installing the software. These entries are necessary for proper operation and accessibility.
> * **POSTMASTER_EMAIL**: The email address specified in `POSTMASTER_EMAIL` must be valid and correctly configured. This email is used by Traefik to issue Let's Encrypt certificates.

Run `vi config/passenger.yml` and paste following content:

```yml
# Database
dsn: /db/passenger.db
# HTTP Read Server
readServer:
  address: 0.0.0.0:8711
  credentials:
    - root/server/read-api
# HTTP Write Server
writeServer:
  address: 0.0.0.0:8712
  credentials:
    - root/server/write-api
# SSH Server
server:
  address: 0.0.0.0:2222
# Vault
vault:
  static:
    path: /db/vault.kdbx
    passwordPath: /config/vault-password
# User Management
users:
  provider: static
```

Run `vi docker-compose.yml` and paste following content:

```yaml
version: "3.6"
services:

  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    network_mode: bridge # bad design, fixme sometime
    command:
      # - "--log.level=DEBUG"
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=${POSTMASTER_EMAIL}"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
      - "8080:8080"
    volumes:
      - "letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  passenger:
    restart: always
    image: dariocalovic/passenger:latest
    network_mode: bridge # bad design, fixme sometime
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.passenger-read.rule=Host(`${READ_FQDN}`)"
      - "traefik.http.routers.passenger-read.entrypoints=websecure"
      - "traefik.http.routers.passenger-read.tls.certresolver=myresolver"
      - "traefik.http.routers.passenger-read.service=passenger-read"
      - "traefik.http.services.passenger-read.loadbalancer.server.port=8711"
      - "traefik.http.routers.passenger-write.rule=Host(`${WRITE_FQDN}`)"
      - "traefik.http.routers.passenger-write.entrypoints=websecure"
      - "traefik.http.routers.passenger-write.tls.certresolver=myresolver"
      - "traefik.http.routers.passenger-write.service=passenger-write"
      - "traefik.http.services.passenger-write.loadbalancer.server.port=8712"
    ports:
      - "${EXPOSED_SSH_PORT}:2222"
    volumes:
      - ./db:/db
      - ./config:/config

volumes:
  letsencrypt:
```

### Step 2: Create the vault password

Run the following command to create a vault password:

Run `echo -n $(openssl rand -base64 32) > config/vault-password`

### Step 3: Pull the Docker image

Pull the Docker images defined in `docker-compose.yml`:

```bash
docker-compose pull
```

**Verification**

When the required images are successfully pulled, the terminal will return the following:  
*Pulling traefik ... done*  
*Pulling passenger ... done*

### Step 4: Start the Passenger Instance

To start the Passenger instance, run:

```bash
docker-compose up -d
```


The terminal will return the following:  
*Creating passenger_passenger_1 ... done*  
*Creating traefik               ... done*

**Verification**

Run the following command to see a list of running containers:

```bash
docker ps -a
```

Run following command, to see if passenger has started:

```bash
docker logs -f passenger_passenger_1
```

You should see following logs:

*2024/07/24 18:50:24 SSH proxy server listening on 0.0.0.0:2222*  
*time="2024-07-24T18:50:24Z" level=info msg="Server listening on 0.0.0.0:8712" endpoint=write*  
*time="2024-07-24T18:50:24Z" level=info msg="Server listening on 0.0.0.0:8711" endpoint=read*

## Usage

Before using Passenger, you need to extract the write and read users from the vault to perform actions.

### Initial Setup

On the initial start, Passenger should have created a file called `/opt/passenger/db/vault.kdbx`. To access this file, follow these steps:

1. Open the `/opt/passenger/db/vault.kdbx` file with KeePass.
2. Retrieve the master password from `/opt/passenger/config/vault-password`.

You should find two entries under `root/server`:

* `write-api`
* `read-api`

### Configuring Authentication Tokens

1. Copy the content of the `write-api` entry.
2. Run the following command to set the `WRITE_AUTH_TOKEN` environment variable (replace username and password with the actual values from KeePass):

  ```bash
  export WRITE_AUTH_TOKEN=$(echo -n 'username:password' | base64 -w 0)
  ``` 
3. Repeat the above steps for the `read-api` entry:

  ```bash
  export READ_AUTH_TOKEN=$(echo -n 'username:password' | base64 -w 0)
  ``` 

Now you can use Passenger in your console session with a web client like `curl`.

### User Management

#### Adding a user

To add a user, you first need to create a credential:

1. Run the following command to create a credential for the user:

    ```bash
    curl --request PUT \
    --url https://write.example.com/credential \
    --header "authorization: Basic $WRITE_AUTH_TOKEN" \
    --data '{
        "identifier": "root/users/tony",
        "provider": "static",
        "data": {
            "username": "tony",
            "password": "secret"
        }
    }'
    ```

2. To verify your newly created credential, run the following command:

    ```bash
    curl --silent --request GET \
    --url https://read.example.com/credential?identifier=root/users/tony \
    --header "authorization: Basic $READ_AUTH_TOKEN" \
    | jq
    ```

3. Add the user by referencing the created credential:

    ```bash
    curl --request PUT \
    --url https://write.example.com/user \
    --header "authorization: Basic $WRITE_AUTH_TOKEN" \
    --data '{
        "identifier": "tony",
        "username": "tony",
        "credential": "root/users/tony",
        "roles": [
            "admin"
        ]
    }'
    ```

4. To verify your newly created user, run the following command:

    ```bash
    curl --silent --request GET \
    --url https://read.example.com/user?identifier=tony \
    --header "authorization: Basic $READ_AUTH_TOKEN" \
    | jq
    ```


### Target Management

#### Adding a Target

To add a target, follow these steps:

1. Create a credential for the target. Note that this credential is used to connect to the target, so the user with this credential needs to exist on the Linux host.

    ```bash
    curl --request PUT \
    --url https://write.example.com/credential \
    --header "authorization: Basic $WRITE_AUTH_TOKEN" \
    --data '{
        "identifier": "root/targets/eclipse/john",
        "provider": "static",
        "data": {
            "username": "clark",
            "password": "secret"
        }
    }'
    ```

2. To verify your newly created credential, run the following command:

    ```bash
    curl --silent --request GET \
    --url https://read.example.com/credential?identifier=root/targets/eclipse/john \
    --header "authorization: Basic $READ_AUTH_TOKEN" \
    | jq
    ```

3. Add the target with the appropriate details. Note that the address should be specified as `host:port` of the target machine. Additionally, ensure that `allowed_ips` includes your remote IP address in CIDR notation, e.g., `YOUR_IP/32`.

    ```bash
    curl --request PUT \
    --url https://write.example.com/target \
    --header "authorization: Basic $WRITE_AUTH_TOKEN" \
    --data '{
        "identifier": "eclipse",
        "address": "192.168.1.21:22",
        "credential": "root/targets/eclipse/john",
        "policy": {
            "reason_required": true,
            "session_max_connections": 2,
            "session_max_seconds": 30,
            "allowed_roles": [
                "admin"
            ],
            "allowed_ips": [
                "80.333.444.55/32"
            ]
        }
    }'
    ```

4. To verify your newly created target, run the following command:

    ```bash
    curl --silent --request GET \
    --url https://read.example.com/target?identifier=eclipse \
    --header "authorization: Basic $READ_AUTH_TOKEN" \
    | jq
    ```

## Login

You have set up your user and target with the necessary credentials. You are now ready to log in to your target via the configured Passenger proxy.

1. Run the following command to initiate the SSH session (replace eclipse@example.com with your target's details):

    ```bash
    ssh tony@eclipse@example.com
    ```

2. Enter the user password you defined in the credential setup.
3. Upon successful password authentication, you will be prompted for a Reason. Enter any reason you like.
4. Passenger will inject the credential and establish a remote connection to the specified target.

Happy Shelling!

## Session Management

### Get All Sessions

To retrieve all sessions, run the following command:

```bash
curl --silent --request GET \
--url https://read.example.com/session \
--header "authorization: Basic $READ_AUTH_TOKEN" \
| jq
```

**Note**: If you do not have `jq` installed, you can omit the `| jq` part of the command. This will output the raw JSON response directly.

## Troubleshoot

Run the following command to view real-time logs from the Passenger container:

```bash
docker logs -f passenger_passenger_1
```

This command will stream the logs to your terminal, allowing you to observe any errors or warnings that may be occurring.
