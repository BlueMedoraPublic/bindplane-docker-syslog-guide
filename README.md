# bindplane-docker-syslog-guide

Guide for using BindPlane logs with Docker + Syslog. This
guide applies to Docker and Docker Swarm.

Logging to Syslog allows all container logs to be centralized
in one location, a drastic improvement over the default
location `/var/lib/docker/containers`.

## Reference Architecture

- Docker or Docker Swarm
- Local syslog with Rsyslog or equivalent

Ubuntu / Debian will use `/var/log/syslog` and RHEL / Centos
will use `/var/log/messages`

## Steps

### Enable Docker's Syslog driver

Docker's syslog driver can be enabled several ways, pick the
option that fits your environment best:
1) daemon.json config file
2) command line options for `docker run`
3) docker-compose yaml configuration (ideal solution)

Docker's official documentation https://docs.docker.com/config/containers/logging/syslog/ [documentation](https://docs.docker.com/config/containers/logging/syslog/)

#### daemon.json
Setting the syslog driver at the configuration file will make
the change global to all containers.

```json
{
  "log-driver": "syslog"
}
```

#### docker run arguments
`--log-driver syslog` can be passed to docker run, allowing only
containers you specify, to log to syslog.

```sh
docker run \
      --log-driver syslog --log-opt syslog-address=udp://1.2.3.4:1111 \
      alpine echo hello world
```

#### docker-compose.yaml ####
Syslog can be enabled for a docker compose / swarm service
using the yaml configuration. See `services.mysql.logging`.

NOTE: The `tag` value will be used to tag logs, which can then
be parsed by the BindPlane log agent or at your logging destination.


```yaml
version: '3.3'
services:
  mysql:
    image: mysql:5.7
    logging:
      driver: syslog
      options:
        tag: mysql
```

### Deploy BindPlane log agent

The BindPlane log agent can be deployed two ways
1) as a container / swarm service
2) as a service outside of Docker

This guide will include instructions on deploying within a
container as part of your Docker compose / swarm stack

#### Docker Compose

```yaml
version: '3.3'
services:
  bindplane-logs-agent:
    image: docker.io/bluemedora/bindplane-log-agent:0.8.0
    deploy:
      mode: global
      resources:
        limits:
          cpus: '0.25'
          memory: 250M
    environment:
      NAME: "{{.Node.Hostname}}"
      COMPANY_ID: ${COMPANY_ID}
      SECRET_KEY: ${BINDPLANE_SECRET_KEY}
      TEMPLATE_ID: ${TEMPLATE_ID}
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers
    logging:
      driver: syslog
      options:
        tag: bindplane-log-agent
```

NOTE: We pin the log agent version to `0.8.0`, however, you should
use the BindPlane web interface to retrieve the latest log agent
version. This can be done going to logs --> agents --> add
--> skip destination --> skip source --> skip group --> kubernetes.

NOTE: The example above uses three environment variables
- COMPANY_ID
- BINDPLANE_SECRET_KEY
- TEMPLATE_ID

You can retrieve these values in the BindPlane web interface
by going to logs --> agents --> add
--> skip destination --> skip source --> skip group --> kubernetes.
