# Guacamole with docker compose and Google SAML2 auth
This is a small documentation how to run a fully working **Apache Guacamole** instance with docker (docker compose) and enable SAML2 authentication with
[Google Workspace](https://support.google.com/a/answer/6087519).

## About Guacamole
Apache Guacamole is a clientless remote desktop gateway. It supports standard protocols like VNC, RDP, and SSH. It is called clientless because no plugins or client software are required. Thanks to HTML5, once Guacamole is installed on a server, all you need to access your desktops is a web browser.

It supports RDP, SSH, Telnet and VNC and is the fastest HTML5 gateway I know. Checkout the projects [homepage](https://guacamole.apache.org/) for more information.

## Prerequisites
You need a working **docker** installation and **docker compose** running on your machine. We assume that the machine
is registered in the DNS as `guacamole.example.com`.

## Google Workspace configuration
Follow the [official documentation](https://support.google.com/a/answer/6087519) to add a custom SAML 2.0 app.
Downlad the IDP metatada file `GoogleIDPMetadata.xml` and set the following parameters.

- **ACS URL**: https://guacamole.example.com/guacamole/api/ext/saml/callback
- **Entity ID**: https://guacamole.example.com

Leave the rest with the default values. If you need groups, map the Google groups you'd like to use in Guacamole with the 
app attribute `groups`.

## Quick start
Clone the GIT repository and start guacamole:

~~~bash
git clone "https://github.com/boschkundendienst/guacamole-docker-compose.git"
cd guacamole-docker-compose
./prepare.sh
cp GoogleIDPMetadata.xml idp-metadata
docker compose up -d
~~~

Your guacamole server should now be available at `https://ip of your server:8443/guacamole`. The default username is `guacadmin` with password `guacadmin`.

## Details
To understand some details let's take a closer look at parts of the `docker-compose.yml` file:

### Networking
The following part of docker-compose.yml will create a network with name `guacnetwork_compose` in mode `bridged`.
~~~python
...
# networks
# create a network 'guacnetwork_compose' in mode 'bridged'
networks:
  guacnetwork_compose:
    driver: bridge
...
~~~

### Services
#### guacd
The following part of docker-compose.yml will create the guacd service. guacd is the heart of Guacamole which dynamically loads support for remote desktop protocols (called "client plugins") and connects them to remote desktops based on instructions received from the web application. The container will be called `guacd_compose` based on the docker image `guacamole/guacd` connected to our previously created network `guacnetwork_compose`. Additionally we map the 2 local folders `./drive` and `./record` into the container. We can use them later to map user drives and store recordings of sessions.

~~~python
...
services:
  # guacd
  guacd:
    container_name: guacd_compose
    image: guacamole/guacd
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/record:rw
...
~~~

#### PostgreSQL
The following part of docker-compose.yml will create an instance of PostgreSQL using the official docker image. This image is highly configurable using environment variables. It will for example initialize a database if an initialization script is found in the folder `/docker-entrypoint-initdb.d` within the image. Since we map the local folder `./init` inside the container as `docker-entrypoint-initdb.d` we can initialize the database for guacamole using our own script (`./init/initdb.sql`). You can read more about the details of the official postgres image [here](http://).

~~~python
...
  postgres:
    container_name: postgres_guacamole_compose
    environment:
      PGDATA: /var/lib/postgresql/data/guacamole
      POSTGRES_DB: guacamole_db
      POSTGRES_PASSWORD: ChooseYourOwnPasswordHere1234
      POSTGRES_USER: guacamole_user
    image: postgres
    networks:
      guacnetwork_compose:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:ro
    - ./data:/var/lib/postgresql/data:rw
...
~~~

#### Guacamole
The following part of docker-compose.yml will create an instance of guacamole by using the docker image `guacamole` from docker hub. It is also highly configurable using environment variables. In this setup it is configured to connect to the previously created postgres instance using a username and password and the database `guacamole_db`. Other settings are used to connect to
the Google Workspace IDP. With the default settings, SAML 2.0 authentication is optional.

~~~python
...
  guacamole:
    container_name: guacamole_compose
    depends_on:
    - guacd
    - postgres
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_HOSTNAME: postgres
      POSTGRES_PASSWORD: 'ChooseYourOwnPasswordHere1234'
      POSTGRES_USER: guacamole_user
      RECORDING_SEARCH_PATH: /record
      SAML_IDP_METADATA_URL: file:///idp-metadata/GoogleIDPMetadata.xml
      SAML_ENTITY_ID: https://guacamole.example.com
      SAML_CALLBACK_URL: https://guacamole.example.com/guacamole
      REMOTE_IP_VALVE_ENABLED: true
      SAML_STRICT: true
      ## Uncomment this line to allow PostgreSQL authentication and optional SAML (default)
      EXTENSION_PRIORITY: '*, saml'
      ## Uncomment this line to allow only SAML authentication
      #EXTENSION_PRIORITY: saml
      SAML_GROUP_ATTRIBUTE: groups
    image: guacamole/guacamole
    networks:
      - guacnetwork_compose
    volumes:
      - ./record:/record:rw
      - ./idp-metadata:/idp-metadata:ro
    ports:
    - 8080:8080/tcp # Guacamole is on :8080/guacamole, not /.
    restart: always...
~~~

## prepare.sh
`prepare.sh` is a small script that creates `./init/initdb.sql` by downloading the docker image `guacamole/guacamole` and start it like this:

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > ./init/initdb.sql
~~~

It creates the necessary database initialization file for postgres.

## reset.sh
To reset everything to the beginning, just run `./reset.sh`.

## WOL

Wake on LAN (WOL) does not work and I will not fix that because it is beyound the scope of this repo. But [zukkie777](https://github.com/zukkie777) who also filed [this issue](https://github.com/boschkundendienst/guacamole-docker-compose/issues/12) fixed it. You can read about it on the [Guacamole mailing list](http://apache-guacamole-general-user-mailing-list.2363388.n4.nabble.com/How-to-docker-composer-for-WOL-td9164.html)

**Disclaimer**

Downloading and executing scripts from the internet may harm your computer. Make sure to check the source of the scripts before executing them!

## Credits
Modified from: https://github.com/boschkundendienst/guacamole-docker-compose