# coreos-vulcand-shipyard
An CoreOS Cluster running Vulcand, Docker Registry, and Shipyard

This project will provide you with a two-tiered CoreOS cluster that provides some infrastructure at one tier,
and a host of Docker-enabled machines managed by Shipyard at the next tier.

# Infrastructure Tier

The infrastructure tier is comprised of two groups of machines: a small etcd cluster and a "devops services" cluster.

## etcd cluster

The etcd cluster is fairly straight forward. It's three etcd machines. There is no step two. Nothing else runs on them,
per the CoreOS team's recommendations for production usage.

Simply edit `cloudformation/devops-etcd.json` to your taste (specifically, specifying your VPC subnets) and then launch
that cluster with CloudFormation.

## Service cluster

The Service cluster, defined in `cloudformation/devops-service.json` file, launches a small cluster of machines that will
run the essential services via fleet. You will need to edit this to reflect your VPC subnets and the IP addresses of your
etcd cluster machines.

### Fleet Units

Once you're up and running here, launch the fleet units.

    $ fleetctl start vulcand.service
    $ fleetctl start docker-registry.service
    $ fleetctl start shipyard.service

### Vulcand Setup

Incoming requests will go through vulcand and be proxied to the correct Docker container. You'll want to point the requisite
DNS entries to whichever machine is hosting the vulcand container. Afterwards, set up backends, servers, and frontends for
the Registry and Shipyard.

For best results, I suggest using vctl, as it will make sure your json is well-formed and includes all of the keys that vulcand
requires.

    $ fleetctl ssh -A -unit=vulcand.service
    coreos-host$ docker exec -ti vulcan1 bash
    vulcand$ vctl backend upsert -id registry
    vulcand$ vctl server upsert -b registry -id registry1 -url http://<<registry host IP>>:5000
    vulcand$ vctl frontend upsert -id registry -b b1 -route 'Host(`your-registry-hostname`) && PathRegexp(`.*`)'
    vulcand$ vctl backend upsert -id shipyard
    vulcand$ vctl server upsert -b shipyard -id shipyard1 -url http://<<shipyard host IP>>:8080
    vulcand$ vctl frontend upsert -id shipyard -b b1 -route 'Host(`your-shipyard-hostname`) && PathRegexp(`.*`)'

# Docker Tier

Launch a bunch of Docker hosts by whatever method you prefer. `cloudformation/docker-tier.json` is an example. Make sure that
`--insecure-registry <<your docker registry>>` is set on each host.

See [Shipyard Docs](http://shipyard-project.com/docs/) for detailed usage.

# TODO

* Rewrite unit files to self-register with vulcand
* Rewrite Docker Tier CloudFormation to self-register with Shipyard