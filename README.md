# Docker Networking v2.0

## Objective

Docker has various features that have networking implications for both the user and the implementation.  These concepts are ports, links, and the networking mode defined by `--net`.  These features have been piecemeal added to Docker to address specific needs.  With Docker Networking v2.0 we intend to pull together these features into a more unified model to give application developers a rich language to describe communication between applications.  As a result of the upgraded networking model, a network will now become a first class object in Docker such that users can create and delete networks and arbitrarily add and remove containers to  the networks.

## Concepts

### Network

"Network" is a very overloaded term.  As such it is important to have a clear definition.  Networks, as defined by Docker, is a loose grouping construct that provides networking connectivity, discoverability, and isolation.  While links provided a unidirectional communication channel between two container, Docker Networks provide a bidirection, many-to-many communication channel between multiple containers.

#### Connectivity

The first key concept of Docker Networks is that they provide connectivity.  This means that all containers in a network should be directly accessible by any other containers in that network.  Docker provides the following guarantees for portability.

1. Any container can access any container using IP (the internet protocol) within a Docker Network.
2. The primary IP address assigned to a container is directly accessible by any other container in a network.

##### Layer 2 vs Layer 3

Docker guarantees layer 3 access using the internet protocol within a network.  This means that a Docker Network is not a direct mapping to layer 2 or even a subnet.  It is a far looser construct than that.  If one was to implement a networking system for Docker, they are still free to tie a network to a specific layer 2 (for example a VLAN) or a subnet.  This is an implementation detail.  From the Docker user perspective they should be unaware of those details.  If the Docker user wanted specific layer 2 functionality, it is still feasible to provide that with the Docker networking model, it will just reduce the portability of the application.  As such that would be discouraged, but Docker does nothing to prevent it.

##### Primary IP address

A container in a network will be provided a primary IP address.  A container may have multiple IP addresses assigned across multiple interfaces, but one IP address per Docker Network will be consider its primary IP address.  If the container communicates this IP to another container in the network, it will be addressable from the other container.  This means that if container A has a primary IP address of 192.168.0.2, container B in the same network will be guaranteed that it can access container A using 192.168.0.2.  This implies that communication within a Docker Network will not go through some IP address translation layer that obscures the actually listening IP of the container.

#### Discoverability

Any container can discover any other container in a Docker Network by using a logical name that is scoped to the Docker Network.  That logical name can be resolved through gethostbyname() and any further name resolution scheme that Docker may use (for example, metadata introspection).

##### Naming

Containers are accessible through DNS using the following format `${ENDPOINT_NAME}.${NETWORK_NAME}.${GLOBAL_DOMAIN}`.
If container A was called `sad_einstein` in the network called `relativity` then container A would be addressable by the DNS name `sad_einstein` or the full qualified domain name (FQDN) `sad_einstein.relativity.${GLOBAL_DOMAIN}`.  Because a container can join more than one network it is possible there could be a naming conflict and as such using the FQDN may be needed.  The global domain name should be configurable and will default to `.internal`.

##### DNS vs /etc/hosts

A container can not assume that the names will be accessible through DNS as the naming implementation may rely on modifications of `/etc/hosts`.  This means that applications should rely on local name resolution as provided by gethostbyname() or similar functions.

#### Isolation

All containers in a Docker Network should by default be isolated from outside network traffic.  In order to allow traffic from outside the network a container must expose a port.

## Ports

Exposing ports is a mechanism used to allow traffic from outside your network into a container.  In the past ports were directly exposed on the host interface.  In the new Docker Networking model ports can be exposed to a network.  A pseudo Docker Network called "host" will exist for backwards compatibility.  At the time of exposing a port a Docker Network can be chosen and that container will be accessible on that network over the exposed ports.  The ports can be mapped such that port 80 on container A can be exposed as 8080.

## Links

Links as they are known today will be deprecated.  They will continue to work as they are currently designed, but will continue to have most of the current limitations in that they are not dynamic or guaranteed to work across hosts.

## Endpoint

Endpoint is an internal concept in Docker but understanding endpoints may help in understanding the naming mechanism used in Docker Networking.  An endpoint is a named instance of an entity in a network.  Two concrete examples of an entity would be a container or an external service.  An external service would be something like a MySQL server that is not managed by Docker, but you would like to access it in a manner as though it was a container managed by Docker.

When an entity is added to a network it is given a name.  For containers, by default that will be the container name, but it could be different.  For example, you could deploy a container called `my-fancy-service-v1`.  You could then add it to a network under the name `my-fancy-service`.  You could then deploy a new container called `my-fancy-servier-v2` but then add it to the network as `my-fancy-service`.  That will effectively deploy a new version of your application.  The key benefit being that both `my-fancy-service-v1` and `my-fancy-service-v2` can be running at the same time but only one will assume the name `my-fancy-service`.

# Backwards Compatibility

Three pseudo networks will exist for backwards compatibility.  They are "none", "host", and "container:id".  These networks will provide the same behavior as `--net none|host|container:id`.  `--net bridge` will now pick the "default" network.  This means that `--net bridge` will not necessarily be the same networking configuration of a Linux bridge but instead the default network chosen at the time.

If a container has joined the "none", "host", or "container:id" networks they will not be allow to join any other networks.  These networks are considered "exclusive" networks and provide advanced namespace oriented networking capabilities that do not fit into the Docker Networking model.

Going forward it will be discouraged for users to use "host" or "container:id" as the functionality and portability of those networks are limited.  They will continue to be supported indefinitely. "container:id" network will have limited name resolution capabilities.  Any container using "container:id" can not rely on the networking naming discoverability defined before.  "host" network decreases portability of a container as it is environment specific.
