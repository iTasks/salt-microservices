Deploying microservice architectures with Salt
==============================================

[![Build Status](https://travis-ci.org/mittwald/salt-microservices.svg?branch=master)](https://travis-ci.org/mittwald/salt-microservices)

Author and copyright
--------------------

Martin Helmich  
Mittwald CM Service GmbH & Co. KG

This code is [MIT-licensed](LICENCE.txt).

Synopsis
--------

This repository contains [Salt](http://saltstack.com) formulas for easily
deploying a small, Docker-based [microservice](http://martinfowler.com/articles/microservices.html)
architecture, implementing the following features:

- Container-based deployment
- Using [Consul][consul] for service discovery
- Using [NGINX][nginx] for load balancing and service routing
- Using [Prometheus][prom] and [Grafana][grafana] for monitoring and alerting
- Zero-downtime (re)deployment of services using a few custom Salt modules

```
            HTTP (Port 80, 443)  │                     │  HTTP (Port 80, 443)
      identity.services.acme.co  │                     │  finances.services.acme.co
                                 ▼                     ▼
                    ┌───────────────────────────────────────┐                       ┌────────┐
                    │                 NGINX                 ├──────────────────────►│        │
                    └────────────┬─────────────────────┬────┘                       │        │
                                 │                     │                            │        │
       HTTP (Port 10000)  ┌──────┴───────┐             │  HTTP (Port 10010)         │        │
identity.services.consul  │              │             │  finances.service.consul   │        │
                          ▼              ▼             ▼                            │        │
                    ┌──────────┐   ┌──────────┐   ┌─────────┐                       │ Consul │
                    │ Identity │   │ Identity │   │ Finance │                       │        │
                    │ Service  │   │ Service  │   │ Service │                       │        │
                    └──────────┘   └──────────┘   └─────────┘                       │        │
                                                                                    │        │
                    ┌───────────────────────────────────────┐                       │        │
                    │                 Docker                │                       │        │
                    └───────────────────────────────────────┘                       └────────┘
```

Motivation
----------

The movitation of this project was to be able to host a small (!) microservice
infrastructure without adding the complexity of additional solutions like Mesos
or Kubernetes.

This probably does not scale beyond a few nodes, but sometimes that's all you
need.

Requirements
------------

-   This formula was tested on Debian Linux. With a bit of luck, it should work
    on recent Ubuntu versions, as well.

-   You need to have [Docker](https://www.docker.com) installed. Since there are
    a number of ways to install Docker, installing Docker is out of scope for
    this formula. If you want to get up and running quickly, use the following
    Salt states for installing Docker:

    ```yaml
    docker-repo:
      repo.managed:
        - name: deb https://apt.dockerproject.org/repo debian-jessie main
        - dist: debian-jessie
        - file: /etc/apt/sources.list.d/docker.list
        - gpgcheck: 1
        - key_url: https://get.docker.com/gpg

    docker-engine:
      pkg.installed:
        - require:
          - repo: docker-repo

    docker:
      service.running:
        - require:
          - pkg: docker-engine
    ```

    Remember to change the *distribution* above from `debian-jessie` to whatever
    you happen to be running.

-   You need the [docker-py](https://github.com/docker/docker-py) Python library.
    The Docker states in this repository require at least version 1.3 of this
    library (tested with 1.3.1). The easiest way to install this package is
    using *pip*:

    ```shellsession
    $ pip install "docker-py>=1.3.0,<1.4.0"
    ```

    Alternatively, use the following Salt states to install docker-py:

    ```yaml
    pip:
      pkg.installed: []
    docker-py:
      pip.installed:
        - name: docker-py>=1.3.0,<1.4.0
        - require:
          - pkg: pip
    ```

Installation
------------

See the [official documentation][salt-formulas] on how to use formulas in your
Salt setup. Assuming you have *GitFS* set up and configured on your Salt master,
simply configure this repository as a GitFS remote in your Salt configuration
file (typically `/etc/salt/master`):

```yaml
gitfs_remotes:
  - https://github.com/mittwald/salt-microservices
```

To quote a very important warning from the official documentation:

>**We strongly recommend forking a formula repository** into your own GitHub
>account to avoid unexpected changes to your infrastructure.

>Many Salt Formulas are highly active repositories so pull new changes with care.
>Plus any additions you make to your fork can be easily sent back upstream with
>a quick pull request!

Using the microservice states
-----------------------------

### Setting up the Salt mine

This formula uses the [Salt mine][salt-mine] feature to discover servers in the
infrastructure. Specifically, the `network.ip_addrs` function needs to be
callable as a mine function. For this, ensure that the following pillar is set
for all servers of your infrastructure:

```yaml
mine_functions:
  network.ip_addrs:
    - eth0
```

### Setting up the Consul server(s)

You need at least one Consul server in your cluster (althoug a number of three
or five should be used in production for a high-availability setup).

First of all, you need to configure a [targeting expression][salt-targeting]
(and optionally a targeting mode) that can be used to match your consul servers.
The default expression will simply match the minion ID against the pattern
`consul-server*`. Use the pillar `consul:server_pattern` for this:

```yaml
consul:
  server_pattern: "<your-consul-server-pattern>"
  server_target_mode: "<glob|pcre|...>"
```

You can install these servers by using the `mwms.consul.server` state in your
`top.sls` file:

```yaml
'consul-server*':
  - mwms.consul.server
```

During the next highstate call, Salt will install a Consul server on each of
these nodes and configure them to run in a cluster. The servers will also be
installed with the Consul UI. You should be able to access the UI by opening
any of your Consul servers on port 8500 in your browser.

### Setting up the Docker nodes

On the Docker nodes, a Consul agent has to be installed (or a Consul server).
You can use the `mwms.consul.agent` state for this. Furthermore, use the
`mwms.services` state to deploy the actual services:

```yaml
'service-node*':
  - mwms.consul.agent
  - mwms.services
```

### Setting up monitoring

For monitoring services, you can use the `mwms.monitoring` state for every node:

```yaml
'service-node*':
  - mwms.monitoring
```

To install a Prometheus server to actually gather metrics, use the
`mwms.prometheus` state on one of your nodes:

```yaml
'service-node-monitoring':
  - mwms.prometheus
```

# Deploying services

## Defining services

Services that should be deployed in your infrastructure are defined using
Salt pillars.

```yaml
#!jinja|yaml|gpg
microservices:
  example:
    hostname: example.services.acme.co
    ssl_certificate: /etc/ssl/certs/your-domain.pem
    ssl_key: /etc/ssl/private/your-domain.key
    ssl_force: True
    containers:
      web:
        instances: 2
        docker_image: docker-registry.acme.co/services/example:latest
        stateful: False
        http: True
        http_internal_port: 8000
        base_port: 10000
        links:
          database: db
        volumes:
          - ["persistent", "/var/lib/persistent-app-data", "rw"]
        environment:
          DATABASE_PASSWORD: |
            -----BEGIN PGP MESSAGE-----
            ...
      db:
        instances: 1
        docker_image: mariadb:10
        stateful: True
        volumes:
          - ["database", "/var/lib/mysql", "rw"]
```

Supposing that each service pillar resides in it's own file (like for example,
`<pillar-root>/services/example.sls`), you can manually distribute your services
on your hosts in your top file:

```yaml
service-node*:
  - services.identity
service-node-001:
  - services.finance
service-node-002:
  - services.mailing
```

**Note**: Yes, you need to distribute services manually to your hosts. Yes, this
is done on purpose to offer a low-complexity solution for small-scale
architectures. If you want more, use [Kubernetes][kubernetes] or
[Marathon][marathon].

## Updating existing services

For updating existing services, you can use the `microservice.redeploy` Salt
module. This module will try to pull a newer version of the image that the
application containers were created from. If a newer image exists, this module
will delete and re-create the containers from the newer image:

```shellsession
$ salt-call microservice.redeploy example
```

This is done sequentially and with a grace time of 60 seconds. If your service
consists of more than one instance of the same container, the deployment will
not cause ~~any~~ significant downtime.

# Reference

## Configuration reference

All deployed microservices are configured using Salt pillars. The root pillar
is the `microservices` pillar. The `microservices` pillar is a map of *service
definitions*. Each key of this map will be used as service name.

### Service definition

A service definition is a YAML object consisting of the following properties:

*   `hostname` (**required**): Describes the public hostname used to address
    this service. This especially important when the service exposes a HTTP API
    or GUI; in this case NGINX  will be configured to use this hostname to route
    requests to respective containers.

*   `containers` (**required**): A map of *container definitions*. Each key of
    this map will be as container name.

*   `ssl_certificate` and `ssl_key` (*optional*): The path to a SSL certificate
    and the associated private key to use to encrypt the connections to the
    respective service.

*   `ssl_force` (*optional*): When a `ssl_certificate` is defined, by default
    all unencrypted HTTP traffic will be redirected to SSL. Set this value to
    `False` to still allow unencrypted HTTP traffic.

*   `checks` (*optional*): Can contain additional health checks for
    [Consul][consul]. See the [Consul documentation on health checks][consul-checks]
    for more information.

*   `check_url` (*optional*): The URL to use for the Consul health check. This
    option will only be used when the service is accessibly via HTTP. If not
    specified, the service's `hostname` property will be used as URL.

### Container definition

A container definition is a YAML object consisting of the properties defined
below. Each container definition may result in *one or more* actual Docker
containers being created (the exact number of containers depends on the
`instances` property). Each container will follow the naming pattern
`<service-name>-<container-name>-<instance-number>`. This means that the example
from above would create three containers:

1. `example-web-0`
2. `example-web-1`
3. `example-db-0`

You can use the following configuration options for each container:

*   `instances` (*default:* `1`): How many instances of this container to spawn.

*   `docker_image` (**required**): The Docker image to use for building this
    container. If the image is not present on the host, the `mwdocker` Salt
    states will try to pull this image.

*   `stateful` (*default:* `False`): You can set this property to `True` if any
    kind of persistent data is stored inside the container (usually, you should
    try to avoid this, for example by using host directories mounted as
    volumes). If a container is as *stateful*, the Salt states will not delete
    and re-create this when a newer version of the image is present.

*   `http` (*default:* `False`): Set to `True` when this container exposes a
    HTTP service. This will cause a respective NGINX configuration to be created
    to make the service be accessible by the public.

*   `http_internal_port` (*default:* `80`): Set this property to the port that
    the HTTP service listens on *inside the container*!

*   `base_port` (**required**): Port that container ports should be mapped on
    on the host. Note that this property does not configure one port, but rather
    the beginning of a port range of `instances` length.

*   `links`: A map of other containers (of the same service) that should be
    linked into this container. The format is `<container-name>: <alias>`.

    Example:

    ```yaml
    links:
      database: db
      storage: data
    ```

*   `volumes`: A list of volumes that should be created for this container. Each
    volume is a list of three items:

    1. The volume name on the host side. **Do not specify an absolute path
       here!** The Salt states will automatically create a new host directory in
       `/var/lib/services/<service-name>/<path>` for you.
    2. The volume path inside the container.
    3. `ro` or `rw` to denote read-only or read&write access.

    Example:

    ```yaml
    volumes:
      - ["database", "/var/lib/mysql", "rw"]
      - ["files", "/opt/service-resources", "ro"]
    ```

*   `volumes_from`: A list of *containers* from which volumes should be used.
    Use the service-local container name, here. Containers should be specified
    as a simple list.

    Example:

    ```yaml
    volumes_from:
      - database
    ```

*   `environment`: A map of environment variables to set for this container.
    **Note:** When using this feature for setting confidential data like API
    tokens or passwords, consider using [Salt's GPG encryption features][salt-gpg].

## SLS reference

### `mwms.consul.server` and `mwms.consul.agent`

These states install a Consul server or agent on a node. The Consul version that
is installed is shipped statically with this formula. The Consul agent will be
run using the [Supervisor process manager][supervisor]

### `mwms.consul.dns`

Configures a node to use the Consul servers as DNS server. This is done by
placing a custom `resolv.conf` file.

### `mwms.services`

This states read the `microservice` pillar (see above) and defines the necessary
states for operating the application containers for these services. This will
include the following:

1. Creating as many Docker containers as defined in the pillar
2. Adjusting the NGINX configuration to make your services accessible to the
   world.
3. Configure maintenance cron jobs for each service as defined in the pillar.
   These will be run in temporary docker containers.

### `mwms.monitoring`

Installs cAdvisor on your host that gathers host and container metrics. This is
probably most useful when you're also using the `mwms.prometheus` state on one
or more of your nodes.

Note that due to Docker issue [#17902](https://github.com/docker/docker/issues/17902),
cAdvisor is started directly on the host, not within a Docker container.

### `mwms.prometheus`

Sets up a Prometheus server for gathering metrics and alerting. The setup
consists of the actual Prometheus service, the Alertmanager and a Grafana
frontend.

## State reference

### `consul.node`

This state registers a new external node using the Consul REST API. This state
requires Consul to be already installed on the node (use one of the
`mwms.consul.*` SLSes for that) and requires the Python [requests][py-requests]
package.

Example:

```yaml
external-node:
  consul.node:
    - address: url-to-external.service.acme.com
    - datacenter: dc1
    - service:
        ID: example
        Port: 80
        Tags:
          - master
          - v1
```

## Module reference

### `microservice.redeploy`

This module will re-deploy a microservice. This module will try to pull a newer
version of the image that the application containers were created from. If a
newer image exists, this module will delete and re-create the containers from
the newer image.

This is done sequentially and with a grace time of 60 seconds. If your service
consists of more than one instance of the same container, the deployment will
not cause ~~any~~ significant downtime.

[cadvisor]: https://github.com/google/cadvisor
[consul]: http://consul.io
[consul-checks]: https://www.consul.io/docs/agent/checks.html
[grafana]: http://grafana.org
[kubernetes]: http://kubernetes.io/
[marathon]: https://mesosphere.github.io/marathon/
[nginx]: http://nginx.org
[prom]: http://prometheus.io
[py-requests]: http://www.python-requests.org/en/latest/
[salt-formulas]: https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html
[salt-gpg]: https://docs.saltstack.com/en/stage/ref/renderers/all/salt.renderers.gpg.html
[salt-mine]: https://docs.saltstack.com/en/stage/topics/mine/index.html
[salt-targeting]: https://docs.saltstack.com/en/latest/topics/targeting/index.html
[supervisor]: http://supervisord.org/
