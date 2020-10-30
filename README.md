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

# Instructions to setup mTLS Communication when using local deployment

In order to apply this configuration when deploying Alfresco and Search Services in a local environment, following steps should be followed.

## Alfresco configuration

For these steps, Alfresco Repository is expected to be installed in `/usr/local/tomcat` folder.

Note that this configuration is only applied from Search Services 2.0.0, as it's using the `classical` configuration from [Alfresco SSL Generator](https://github.com/Alfresco/alfresco-ssl-generator)

Copy the contents of the [keystores/alfresco](keystores/alfresco) folder to `/usr/local/tomcat/alf_data/keystore` folder.

```
$ ls -l /usr/local/tomcat/alf_data/keystore/
keystore
keystore-passwords.properties
ssl.keystore
ssl-keystore-passwords.properties
ssl.truststore
ssl-truststore-passwords.properties
```

Add the following values to your `alfresco-global.properties` file.

```
$ cat /usr/local/tomcat/shared/classes/alfresco-global.properties
solr.host=localhost
solr.port.ssl=8983
solr.secureComms=https
dir.keystore=/usr/local/tomcat/alf_data/keystore
encryption.ssl.keystore.type=JCEKS
encryption.ssl.truststore.type=JCEKS
```

Add the following 8443 Connector to your Tomcat configuration file.

```
$ cat /usr/local/tomcat/conf/server.xml
...
    <Connector port="8443" protocol="org.apache.coyote.http11.Http11Protocol"
        connectionTimeout="20000"
        SSLEnabled="true" maxThreads="150" scheme="https"
        keystoreFile="/usr/local/tomcat/alf_data/keystore/ssl.keystore"
        keystorePass="kT9X6oe68t" keystoreType="JCEKS" secure="true"
        truststoreFile="/usr/local/tomcat/alf_data/keystore/ssl.truststore"
        truststorePass="kT9X6oe68t" truststoreType="JCEKS" clientAuth="want" sslProtocol="TLS">
    </Connector>
  </Service>
</Server>
```

## Search Services configuration

For these steps, Search Services is expected to be installed in `/opt/alfresco-search-services` folder.

Note that this configuration is only applied from Search Services 2.0.0, as it's using the `current` configuration from [Alfresco SSL Generator](https://github.com/Alfresco/alfresco-ssl-generator)

Copy the contents of the [keystores/solr](keystores/solr) folder to `/opt/alfresco-search-services/keystore` folder.

```
$ ls -l /opt/alfresco-search-services/keystore
ssl-repo-client.keystore
ssl-repo-client.truststore
```

Add the following values to your `/opt/alfresco-search-services/solrhome/alfresco/conf/solrcore.properties` file (or to your `/opt/alfresco-search-services/solrhome/templates/rerank/conf/solrcore.properties` file if you are creating cores by default with `-Dcreate.alfresco.defaults=alfresco,archive` command line option)

```
alfresco.secureComms=https
alfresco.encryption.ssl.keystore.location=/opt/alfresco-search-services/keystore/ssl-repo-client.keystore
alfresco.encryption.ssl.keystore.passwordFileLocation=
alfresco.encryption.ssl.keystore.type=JCEKS
alfresco.encryption.ssl.truststore.location=/opt/alfresco-search-services/keystore/ssl-repo-client.truststore
alfresco.encryption.ssl.truststore.passwordFileLocation=
alfresco.encryption.ssl.truststore.type=JCEKS
```

Add the following values to your `/opt/alfresco-search-services/solr.in.sh` file (or to `solr.in.cmd` file if you are installing SOLR in Windows)

```
$ cat /opt/alfresco-search-services/solr.in.sh
SOLR_SSL_TRUST_STORE=/opt/alfresco-search-services/keystore/ssl-repo-client.truststore
SOLR_SSL_TRUST_STORE_TYPE=JCEKS
SOLR_SSL_KEY_STORE=/opt/alfresco-search-services/keystore/ssl-repo-client.keystore
SOLR_SSL_KEY_STORE_TYPE=JCEKS
SOLR_SSL_NEED_CLIENT_AUTH=true
```

Start SOLR using the following parameters:

```
$ /opt/alfresco-search-services/solr/bin/solr start -a
"-Dcreate.alfresco.defaults=alfresco,archive \
-Dsolr.ssl.checkPeerName=false \
-Dsolr.allow.unsafe.resourceloading=true \
-Dsolr.jetty.truststore.password=kT9X6oe68t
-Dsolr.jetty.keystore.password=kT9X6oe68t
-Dssl-keystore.password=kT9X6oe68t
-Dssl-keystore.aliases=ssl-alfresco-ca,ssl-repo-client
-Dssl-keystore.ssl-alfresco-ca.password=kT9X6oe68t
-Dssl-keystore.ssl-repo-client.password=kT9X6oe68t
-Dssl-truststore.password=kT9X6oe68t
-Dssl-truststore.aliases=ssl-alfresco-ca,ssl-repo,ssl-repo-client
-Dssl-truststore.ssl-alfresco-ca.password=kT9X6oe68t
-Dssl-truststore.ssl-repo.password=kT9X6oe68t
-Dssl-truststore.ssl-repo-client.password=kT9X6oe68t" -f
```
