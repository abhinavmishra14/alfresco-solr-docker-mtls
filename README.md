# Alfresco Search Services with mTLS configuration

Deployment template based in official [Docker Composition](https://github.com/Alfresco/acs-community-deployment/tree/master/docker-compose) using mTLS communication between SOLR and Alfresco Repository.

Alfresco Repository is using the `classical` certificates format, while SOLR is using the `current` certificates format. More details available in [https://github.com/Alfresco/alfresco-ssl-generator#parameters](https://github.com/Alfresco/alfresco-ssl-generator#parameters)

You should review volumes, configuration, modules & tuning parameters before using this composition in **Production** environments.

## Source Images

* [alfresco-content-repository-community:6.2.1-A8](https://hub.docker.com/r/alfresco/alfresco-content-repository-community)
* [alfresco-share:6.2.1](https://hub.docker.com/r/alfresco/alfresco-share)
* [alfresco-search-services:2.0.0.1](https://hub.docker.com/r/alfresco/alfresco-search-services)
* [postgres:11.7](https://hub.docker.com/_/postgres)
* [angelborroy/acs-proxy:1.0.0](https://hub.docker.com/repository/docker/angelborroy/acs-proxy)

## Project structure

```
.
├── .env
├── alfresco
│   └── Dockerfile
├── config
│   └── nginx.htpasswd
├── docker-compose.yml
├── keystores
│   ├── alfresco
│   │   ├── keystore
│   │   ├── keystore-passwords.properties
│   │   ├── ssl-keystore-passwords.properties
│   │   ├── ssl-truststore-passwords.properties
│   │   ├── ssl.keystore
│   │   └── ssl.truststore
│   ├── client
│   │   └── browser.p12
│   └── solr
│       ├── ssl-repo-client.keystore
│       └── ssl-repo-client.truststore
└── search
    └── Dockerfile
```

* `.env` includes Docker environment variables to set Docker Image release numbers
* `alfresco` folder includes configuration for ACS Repository Docker Image
* `config` NGINX configuration to set the SOLR Admin Web Console user and password credentials
* `docker-compose.yml` is a Docker Compose template to use ACS Community with mTLS Communication
* `keystores` folder includes keystore and truststores files for Alfresco Repository (classic format, with password files) and SOLR (current format, without password files)

## SOLR Considerations

Alfresco SOLR API has been protected to be accessed from outside Docker network, as using HTTP allows unauthenticated requests.

```
    # Protect access to SOLR APIs
    location ~ ^(/.*/service/api/solr/.*)$ {return 403;}
    location ~ ^(/.*/s/api/solr/.*)$ {return 403;}
    location ~ ^(/.*/wcservice/api/solr/.*)$ {return 403;}
    location ~ ^(/.*/wcs/api/solr/.*)$ {return 403;}

    location ~ ^(/.*/proxy/alfresco/api/solr/.*)$ {return 403 ;}
    location ~ ^(/.*/-default-/proxy/alfresco/api/.*)$ {return 403;}
```

SOLR Web Console access has been protected with username/password (admin/admin).


# How to use this composition

## Start Docker

Start docker and check the ports are correctly bound.

```bash
$ docker-compose up -d
$ docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'
proxy_1               angelborroy/acs-proxy:1.0.0               80/tcp, 0.0.0.0:8080->8080/tcp
solr6_1               alfresco-solr-docker-mtls_solr6	          10001/tcp, 0.0.0.0:8083->8983/tcp
share_1               alfresco/alfresco-share:6.2.1             8000/tcp, 8080/tcp
activemq_1            alfresco/alfresco-activemq:5.15.8         0.0.0.0:5672->5672/tcp, ...
postgres_1            postgres:11.7                             0.0.0.0:5432->5432/tcp
alfresco_1            alfresco-solr-docker-mtls_alfresco        8080/tcp, 0.0.0.0:8443->8443/tcp
transform-core-aio_1 alfresco/alfresco-transform-core-aio:2.3.5 0.0.0.0:8090->8090/tcp
```

### Viewing System Logs

You can view the system logs by issuing the following.

```bash
$ docker-compose logs -f
```

Logs for every service are also available at `logs` folder.

## Access

Use the following username/password combination to login.

 - User: admin
 - Password: admin

Alfresco and related web applications can be accessed from the below URIs when the servers have started.

```
http://localhost:8080/alfresco      - Alfresco Repository
http://localhost:8080/share         - Alfresco Share
https://localhost:8083/solr         - Alfresco Search Services (use keystores/client/browser.p12 certificate)
```
