# Hung's Docker / Swarm / Kubernetes learning notes

## Quickest way to install Docker
-   `curl -sSL https://get.docker.com/ | sh`: Docker's automated script to add their repository and install all dependencies

## Concepts
-   Port conflict errors occur when multiple services are running on the same published port on the same host, not in multiple containers. Each container gets its own internal IP behind NAT, so the ports won't conflict across two container network interfaces.

## Very basics
-   `docker container run $CONTAINER_NAME` - start a new container
-   `docker container ls -a` - list all containers (running and stopped)
-   `docker image ls` - list all images
-   `docker container run -p $HOST:$CONTAINER` - port forwarding (`-p` is short for `--publish`)

## Getting a Shell Inside Containers
-   `docker container run -it` - start new container interactively
-   `docker container exec -it` - run additional commend in existing container
-   By using the `docker exec -it <container> sh` (or bash) command on a container, we can connect to a shell from inside it.

## Container networking
### Concepts
-   Create your apps so frontend/backend sit on same Docker network.
-   Their inter-communication never leaves host
-   All externally exposed prots closed by default

#### DNS
-   Containers shouldn't rely on IPs for inter-communication.
-   DNS for friendly names is built-in if you use custom networks.

### Commands
-   `docker network ls` Show networks
-   `docker network inspect` Inspect a network
-   `docker network create --driver` Create a network
-   `docker network connect` Attach a network to container
-   `docker network disconnect` Detach a network from container

## Docker *Image*
### Concepts
-   Images are made of file system changes and metadata
-   Each layer is uniquely identified and only stored once on a host
-   This saves storage space on host and transfer time on push/pull
-   A container is just single read/write layer on top of image
-   **tagging** is important, it identifies a specific commit/"version" of an image

### Dockerfile
#### Stanzas
-   The main purpose of a `CMD` is to provide defaults for an executing container.
The CMD instruction has three forms:

    -   `CMD ["executable","param1","param2"]` (exec form, this is the preferred form)
    -   `CMD ["param1","param2"]` (as default parameters to ENTRYPOINT)
    -   `CMD command param1 param2` (shell form)

#### Building
-   Keeping the parts that change the least **AT THE TOP** of the Dockerfile, the parts that change the most **AT THE BOTTOM** of the Dockerfile.

### Persistent data
-   Docker has 2 main methods to persist data: **volumes** and **bind mounts**.
-   **Volume**: e.g. `docker container run -d --name mysql MYSQL_ALLOW_EMPTY_PASSWORD=True -v mysql-db:/var/lib/mysql mysql`. Whereby `mysql-db` is the named volume.
-   **Bind mounting**: 
    -   This maps a host file or directory to a container file or directory (If they both exists, the host wins).
    -   Can't use in `Dockerfile`, must be at `container run ...`
    -   E.g. `container run ... -v /user/hung/stuff:/path/container`
    -   Changes are updated **live!**. If a file is edited/added/removed from the host bind-mounted folder, the change will be reflected in the container! 

### docker-compose

-   Volumes: add `:ro` at the end of the volume path to specify "Read-only" e.g. `docker container run ... -v /local/path:/container/path:ro ...`
-   **docker-compose** will automatically create a virtual network to include all the services.
-   What you name the service under `services` will actually be the **DNS** on the Docker container private network!

### Swarm

-   `docker swarm init` to enable initialise Swarm. Swarm is not activated out of the box for Docker.
-   Key commands:
    -   `docker service create ...`
    -   `docker service update ...`
    -   `docker service ls ...`
    -   `docker service rm ...`
    -   `docker service ps <service_name>`: List the tasks of one or more services
-   Adding nodes to a swarm: 
    -   The nodes need to be able to talk to each other (ICMP, TCP and UDP connections enabled).
-   Docker Swarm networking:
    -   `docker network create --driver overlay <network_name>`: overlay network driver is used for container communication across a swarm.
    -   Containers and networks are a many-to-many relationship. A single container can be attached to many networks and vice versa.
    -   Swarm has **routing mesh**, which receives packets for a Service to distribute to proper Tasks. It spans all nodes across the Swarm and creates a virtual IP (VIP) serving as an interface for inbound traffic. This IP is also a load balancer that decides which node gets the traffic.
-   Node management:
    -   `docker node ls`: List out all the nodes.
    -   `docker swarm update ...`: Update certain characteristics of the Swarm environment.

#### Stacks
-   Stacks are Production Grade Compose, accepting Compose files as their declarative definition for services, networks, and volumes.
-   `docker stack deploy ...` command

-   Version needs to be 3+

#### Secrets
-   Secrets are applicable in Swarm only.
-   `docker secret create secret_name secret_file`: Creating secrets from plain text files
-   `echo "yourSecret" | docker secret create secret_name -`: Creating secrets from typing it out
-   After *removing/updating* secrets, service containers will be re-created. It is part of the immutable design.
-   Secrets and Stacks: version needs to be at least 3.1

#### Full App Lifecycle With Compose
-   Single set of Compose files for:
    -   Local `docker-compose up` development environment
    -   Remote `docker-compose up` CI environment
    -   Remote `docker stack deploy` production environment
    -   `docker-compose -f a.yml -f b.yml up`: Layering Compose files:
        -   docker-compose.yml: base file
        -   docker-compose.override.yml: override file, Docker Compose will read this file automatically
        -   docker-compose.test.yml: test compose file for CI environment
        -   docker-compose.prod.yml: compose file to bring into production
        -   Examples:
            -   CI deployment: `docker-compose -f docker-compose.yml -f docker-compose.test.yml up -d`
            -   Getting production config yml: `docker-compose -f docker-compose.ynl -f docker-compose.prd.yml config > output.yml`
    -   *Remember*: `docker compose` is only for local development, not production!

#### Swarm Update Examples
-   Just update the image used to a newer version: `docker service update --image myapp:1.2.1 <servicename>`
-   Adding an environment variable and remove a port: `docker service update --env-add NODE_ENV=production --publish-rm 8080`
-   Change number of replicas of two services: `docker service scale <servicename1>=8 <servicename2>=6`
-   Swarm Updates in Stack Files:
    -   Same command. Just edit the YAML file, then `docker stack deploy -c file.yml <stackname>`
-   `docker service update --force <servicename>`: Force update even if no changes require it

#### Docker Healthchecks
-   Healthcheck status shows up in `docker container ls`
-   Check last 5 healthchecks with `docker container inspect`
-   Docker run does nothing with healthchecks
-   Services will replace tasks if they fail healthcheck
-   Service updates wait for them before continuing
-   Example command:
    ```shell
    docker run \
        --health-cmd="curl -f localhost:9200/_cluster/health || false" \
        --health-interval=5s \
        --health-retries=3 \
        --health-timeout=2s \
        --health-start-period=15s \
        elasticsearch:2
    ```
-   Example in Dockerfile - Nginx:
    ```Dockerfile
    FROM nginx:1.13

    HEALTHCHECK --interval=30s --timeout=2s --interval=3s --retries=3 \
        CMD curl -f http://localhost/ || exit 1
    ```
-   Example in Dockerfile - Postgres:
    ```Dockerfile
    FROM postgres

    # specify real user with -U to prevent errors in logs

    HEALTHCHECK --interval=5s --timeout=3s \
        CMD pg_isready -U postgres || exit 1
    ```

#### Other notes
-   When we do a `docker stack deploy` on an existing stack, it will deploy the changes as service updates.