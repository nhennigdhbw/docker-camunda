# SWIM - Camunda BPM Docker Image

This Camunda BPM community project provides docker images of the latest Camunda
BPM platform releases. The images can be used to demonstrate and test the
Camunda BPM platform or can be extended with own process applications. It is
planned to provide images on the official [docker registry][] for every upcoming
release, which includes alpha releases.

## Get started

To start the latest release:

```
docker pull camunda/camunda-bpm-platform:latest
docker run -d --name camunda -p 8080:8080 camunda/camunda-bpm-platform:latest
```

### Tasklist, Cockpit, Admin Web Apps

The three Camunda webapps are accessible through the landing page: http://localhost:8080/camunda-welcome/index.html

The default credentials for admin access to the webapps is:

- Username: `demo`
- Password: `demo`

### Rest-API

The Camunda Rest-API is accessible through: http://localhost:8080/engine-rest

See the [Rest-API](https://docs.camunda.org/manual/latest/reference/rest/)
documentation for more details on how to use it.

**Note**: The Rest-API does not require authentication by default. Please
follow the instructions from the [documentation](https://docs.camunda.org/manual/latest/reference/rest/overview/authentication/)
to enable authentication for the Rest-API.

## Java Versions

Our docker images are using the latest LTS OpenJDK version supported by
Camunda BPM. This currently means:

 - Camunda 7.12 will be based on OpenJDK 11
 - All previous versions are based on OpenJDK 8

While all the OpenJDK versions supported by Camunda will work, we will not
provide a ready to use image for them.

### Java Options

To override the default Java options the environment variable `JAVA_OPTS` can
be set. The default value is set to limit the heap size to 768 MB and the
metaspace size to 256 MB.

```
JAVA_OPTS="-Xmx768m -XX:MaxMetaspaceSize=256m"
```

### Use docker memory limits

Instead of specifying the Java memory settings it is also possible to instruct
the JVM to respect the docker memory settings. As the image uses Java 8 it has
to be enabled using the `JAVA_OPTS` environment variable. Using the following
settings the JVM will respect docker memory limits specified during startup.

```
JAVA_OPTS="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap"
```

## Database environment variables

The used database can be configured by providing the following environment
variables:

- `DB_CONN_MAXACTIVE` the maximum number of active connections (default: `20`)
  - for `tomcat`, this is internally mapped to the `maxTotal` configuration property.
- `DB_CONN_MAXIDLE` the maximum number of idle connections (default: `20`)
  - ignored when app server = `wildfly`
- `DB_CONN_MINIDLE` the minimum number of idle connections (default: `5`)
- `DB_DRIVER` the database driver class name, supported are h2, mysql, postgresql and oracle:
  - h2: `DB_DRIVER=org.h2.Driver`
  - mysql: `DB_DRIVER=com.mysql.jdbc.Driver`
  - postgresql: `DB_DRIVER=org.postgresql.Driver`
  - oracle: `DB_DRIVER=oracle.jdbc.OracleDriver`
- `DB_URL` the database jdbc url
- `DB_USERNAME` the database username
- `DB_PASSWORD` the database password
- `DB_VALIDATE_ON_BORROW` validate database connections before they are used (default: `false`)
- `DB_VALIDATION_QUERY` the query to execute to validate database connections (default: `"SELECT 1"`)
  - oracle: `DB_VALIDATION_QUERY="select 1 from dual"`
- `DB_PASSWORD_FILE` this supports [Docker Secrets](https://docs.docker.com/engine/swarm/secrets/). 
  Put here the path of the secret, e.g. `/run/secrets/camunda_db_password`. 
  Make sure that `DB_PASSWORD` is not set when using this variable!
- `SKIP_DB_CONFIG` skips the automated database configuration to use manual
  configuration
- `WAIT_FOR` wait for a `host:port` to be available over TCP before starting
- `WAIT_FOR_TIMEOUT` how long to wait for the service to be avaiable - defaults to 30 seconds

For example to use a postgresql docker image as database you can start the
platform as follows:

```
# start postgresql image with database and user configured
docker run -d --name postgresql ...

docker run -d --name camunda -p 8080:8080 --link postgresql:db \
           -e DB_DRIVER=org.postgresql.Driver \
           -e DB_URL=jdbc:postgresql://db:5432/process-engine \
           -e DB_USERNAME=camunda \
           -e DB_PASSWORD=camunda \
           -e WAIT_FOR=db:5432 \
           camunda/camunda-bpm-platform:latest
```

Another option is to save the database config to an environment file, i.e.
`db-env.txt`:

```
DB_DRIVER=org.postgresql.Driver
DB_URL=jdbc:postgresql://db:5432/process-engine
DB_USERNAME=camunda
DB_PASSWORD=camunda
WAIT_FOR=db:5432
```

and use this file to start the container:

```
docker run -d --name camunda -p 8080:8080 --link postgresql:db \
           --env-file db-env.txt camunda/camunda-bpm-platform:latest
```

The docker image already contains drivers for `h2`, `mysql` and `postgresql`.
If you want to use other databases you have to add the driver to the container
and configure the database settings manually by linking the configuration file
into the container.

To skip the configuration of the database by the docker container and use your
own configuration set the environment variable `SKIP_DB_CONFIG` to a non
empty value:

```
docker run -d --name camunda -p 8080:8080 -e SKIP_DB_CONFIG=true \
           camunda/camunda-bpm-platform:latest
```

## Waiting for database

Starting the Camunda BPM Docker image requires the database to be already available.
This is quite a challenge when the database and the Camunda BPM are both docker containers spawned simualtenously eg. by `docker-compose` or inside a Kubernetes Pod.
To help with that, the Camunda BPM Docker image includes [wait-for-it.sh](https://github.com/vishnubob/wait-for-it) to allow the container to wait until a 'host:port' is ready.
The mechanism can be configured by two environment variables:

- `WAIT_FOR`: the service `host:port` to wait for
- `WAIT_FOR_TIMEOUT`: how long to wait for the service to be available in seconds

Example with a PostgreSQL container:

```
docker run -d --name postgresql ...

docker run -d --name camunda -p 8080:8080 --link postgresql:db \
           -e DB_DRIVER=org.postgresql.Driver \
           -e DB_URL=jdbc:postgresql://db:5432/process-engine \
           -e DB_USERNAME=camunda \
           -e DB_PASSWORD=camunda \
           -e WAIT_FOR=db:5432 \
           -e WAIT_FOR_TIMEOUT=60 \
           camunda/camunda-bpm-platform:latest
```

## Volumes

The Camunda BPM Platform is installed inside the `/camunda` directory. Which
means the tomcat configuration files are inside the `/camunda/conf/` directory
and the deployments on tomcat are in `/camunda/webapps/`. The directory
structure depends on the application server.

## Debug

To enable JPDA inside the container you can set the environment variable
`DEBUG=true` on startup of the container. This will allow you to connect to the
container on port `8000` to debug your application.

## Prometheus JMX Exporter

To enable Prometheus JMX Exporter inside the container you can set the environment 
variable `JMX_PROMETHEUS=true` on startup of the container. 
This will allow you to get metrics in Prometheus format at `<host>:9404/metrics`. 
For configuring exporter you need attach your configuration as a container volume 
at `/camunda/javaagent/prometheus-jmx.yml`.

### Extend Docker Image

As we release these docker images on the offical [docker registry][] it is
easy to create your own image. This way you can deploy your applications
with docker or provided an own demo image. Just specify in the `FROM`
clause which Camunda image you want to use as a base image:

```
FROM camunda/camunda-bpm-platform:tomcat-latest

ADD my.war /camunda/webapps/my.war
```


### Change timezone

To change the timezone of the docker container you can set the environment variable `TZ`.

```
docker run -d --name camunda -p 8080:8080 \
           -e TZ=Europe/Berlin \
          camunda/camunda-bpm-platform:latest
```


## Branching Model (not applicable)

Branches and their roles in this repository:

- `next` is the branch where new features go into (default branch)
- `master` should get only changes needed to support the current `master` of [https://github.com/camunda/camunda-bpm-platform](camunda-bpm-platform) repositories
- `7.x` branches get created from `master` when a Camunda BPM minor release happened and only then `next` is merged into `master` once
