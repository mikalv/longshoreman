![Longshoreman](http://i.imgur.com/4vkVHdI.png)

# Longshoreman

Longshoreman automates application deployment using Docker. Just create a Docker repository (or use a service), configure the cluster using AWS or Digital Ocean (or whatever you like) and deploy applications using a Heroku-like CLI tool.

[Main GitHub project page](https://github.com/longshoreman)

## Why make this?

We created Longshoreman because we love using Docker but were frustrated with the lack of production-ready deployment options that were available at the time. We looked closely at Deis, Flynn, Dokku and others, but they either did not meet our requirements or were explicitly marked as not ready for production. We were extremely impressed by Deis in particular and its use of bleeding edge technologies like CoreOS, etcd and systemd. The biggest shortcoming we found with Deis is that it rebuilds
Dockerfiles from scratch for each deploy (as far as I know).

## Who made it?

Longshoreman is sponsored/developed by [Wayfinder](http://wayfinder.co). We're currently using it in production to orchestrate the deployment of microservices (which it suits very nicely).

## How does it work?

The Longshoreman service has 2 main components, a router and a controller, which live in the same application instance. It also uses a Docker registry and Redis as its configuration database.

[How to setup a Longshoreman cluster in 10 minutes](http://mikejholly.com/create-your-own-heroku-in-10-minutes/)

### Controller

The Longshoreman controller is a service which orchestrates the deployment of Docker applications across a cluster and controls how traffic (web or what have you) is routed to individual application instances. It communicates over HTTP with the CLI tool. Launching a new version of an application is as simple as `longshoreman --app my.app.com deploy docker.repo.com/image:tag`. Your application will be deployed to 2 or more nodes (depending on the size of your cluster and its available resources). Versioning and rollbacks can be achieved using image tags.

### Routers

The routers dynamically direct incoming web traffic to the correct application instances. They are simple Node.js reverse proxies that pass requests on to the underlying application instances. The Router runs on the same machine as the controller. Traffic hits the Controller application only if the incoming Host header matches your `CONTROLLER_HOST` environmental variable.

### CLI

The command line tool is an interface to the Longshoreman controller service. It allows users to describe the state of the application cluster, deploy new instances of applications (with zero-downtime), add and remove hosts, add and remove application environmental variables and more. See the link below for full documentation.

[CLI Repository](https://github.com/longshoreman/cli)

### Other Components

#### Registry

Longshoreman uses a Docker registry (ideally a private one) to coordinate application versioning and deployment. Docker registries are outside of the scope of this project, so if you're unfamiliar with them please [read more here](https://github.com/dotcloud/docker-registry). You will need a Docker registry (most likely a private one) to use Longshoreman. There are several private registry hosting
companies (including Docker proper) that provide this service for a low cost. You can also host a registry yourself (preferred). Setting up an S3-backed private registry is fairly simple. Just follow [these instructions](https://github.com/dotcloud/docker-registry).

#### Configuration Store

We are currently using Redis to store and distribute the cluster's state. Longshoreman uses PubSub to notify the cluster of updates to the internal application routing table. We're looking into support for etcd as a single point of failure exists if the Redis instance is not redundant.

## Quick Start

This guide will walk you through creating a Longshoreman powered cluster (we're using EC2 running Ubuntu in this example).

To create an application cluster using Longshoreman, you'll need at least 1 server node but 3 or more work best if you want redundancy.  

### 1. Deploy container nodes

1. Launch 1 or more EC2 instances.
1. Install Docker (http://docs.docker.com/installation/)
1. Edit the Docker config `/etc/default/docker`. Set `DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"` to enable the Docker Remote API
1. Restart Docker with `sudo service docker.io restart`
1. Start the controller with `sudo docker run -d -p 80:80 -e REDIS_HOST=$REDIS_HOST -e REDIS_PORT=6379 -e CONTROLLER_HOST=$CONTROLLER_HOST longshoreman/longshoreman`

Where `$REDIS_HOST` is set to your Redis hostname or IP and `$CONTROLLER_HOST` is your desired Longshoreman controller location (e.g., lsm.domain.com).

### 2. Configure and deploy applications using the CLI

1. Run `longshoreman init` to configure your credentials. Enter the Longshoreman controller domain and your token. The token is auto-generated and is stored in Redis (`GET token`).
1. `longshoreman hosts:add <host-ip>` to make Longshoreman aware of your nodes.
1. `longshoreman apps:add my.app.domain` to add a new service or application to your cluster.
1. `longshoreman --app my.app.domain envs:set FOO=bar` to configure your application's runtime settings.
1. `longshoreman --app my.app.domain deploy my.docker.reg/repo:tag` to deploy the first version of your application.
1. Point your domain to your load balancer's CNAME and Bob's your uncle.

Check out [the CLI repository](https://github.com/longshoreman/cli) for full documentation.

## Using SSL

We currently recommend using something like ELB where SSL termination happens on the load balancer.

## License

MIT
