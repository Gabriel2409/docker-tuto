# Table of contents

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=2 orderedList=false} -->

<!-- code_chunk_output -->

- [Table of contents](#table-of-contents)
- [Docker - where is it on host ?](#docker-where-is-it-on-host)
- [Containers](#containers)
  - [Cheat sheet](#cheat-sheet)
  - [Check installation works](#check-installation-works)
  - [Useful commands](#useful-commands)
  - [Publish port](#publish-port)
  - [Bind mount](#bind-mount)
  - [Limit resources](#limit-resources)
  - [Ownership](#ownership)
  - [Basic docker container commands (details)](#basic-docker-container-commands-details)
- [Images](#images)
  - [File system](#file-system)
  - [Union filesystem](#union-filesystem)
  - [Copy on write](#copy-on-write)
  - [Dockerfile](#dockerfile)
  - [Build an image](#build-an-image)
  - [List images](#list-images)
  - [Multistage build](#multistage-build)
  - [Cache](#cache)
  - [Build context](#build-context)
  - [Basic docker image commands (details)](#basic-docker-image-commands-details)
- [Registry](#registry)
  - [How to use](#how-to-use)
  - [Providers:](#providers)
  - [Docker Hub](#docker-hub)
  - [Docker Open source registry](#docker-open-source-registry)
- [Storage](#storage)
  - [Containers and data persistence](#containers-and-data-persistence)
  - [Volumes](#volumes)
  - [Volume drivers](#volume-drivers)
  - [Plugins](#plugins)
- [Docker machine (DEPRECATED)](#docker-machine-deprecated)
- [Docker compose](#docker-compose)
  - [docker-compose.yml](#docker-composeyml)
  - [docker-compose binary](#docker-compose-binary)
  - [service discovery](#service-discovery)
  - [Voting app](#voting-app)
- [Docker Swarm](#docker-swarm)
  - [Swarm mode](#swarm-mode)
  - [Raft: consensus in swarm mode](#raft-consensus-in-swarm-mode)
  - [Node](#node)

<!-- /code_chunk_output -->

# Docker - where is it on host ?

- `/var/lib/docker` for linux
- in wsl go to `\\wsl$\docker-desktop-data\version-pack-data\community\docker`

# Containers

## Cheat sheet

More details in the sections after

### Basic docker container commands

- `run`: creates a container
- `ls`: lists containers
- `inspect`: details of a container
- `logs`: get logs
- `exec`: lauch a process in an existing container
- `stop`: stops a container
- `rm`: removes a container
- `commit`: creates an image from the container

Note : launch shell in vm : `docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh`

### docker container run argument:

- `-t` : allocates pseudo TTY
- `-i` : keep STDIN open (interactive)
- `-d` : detached mode
- `--rm` : clean up on exit
- `--restart=on-failure` : auto restart
- `-p HOST_PORT:CONTAINER_PORT`: port mapping
- `-v HOST_PATH:CONTAINER_PATH` : mounting
- `--user username` : specifies user
- `--cpus 0.5` : limits to 0.5 cpu
- `--memory 32m` : kills process after memory exceeded

### Dockerfile main instructions

- `FROM`: defines the base image whose filesystem we want to enrich
- `ENV`: defines env variables
- `RUN`: Runs a command, builds image fs
- `COPY/ADD`: add resources from host machine to image file system
- `EXPOSE`: exposes a port of the application
- `HEALTHCHECK`: checks the health of application
- `VOLUME`: defines a directory in the fs of the image whose content will be managed from outside the container
- `WORKDIR`: defines the working directory
- `USER`: defines the user used to launch the app
- `ENTRYPOINT`: defines command executed when container is launched
- `CMD`: defines default args that will be fed to the entrypoint

Summary of discussion CMD vs ENTRYPOINT: https://stackoverflow.com/questions/21553353/what-is-the-difference-between-cmd-and-entrypoint-in-a-dockerfile

### Basic docker container image commands:

- `pull` : gets image from registry
- `push`: push image to registry
- `inspect`: details of an image
- `history`: shows history of layers
- `ls`: shows local images
- `save`: saves an image to a .tar => `docker save -o file.tar image`
- `load`: loads image from .tar => `docker load < file.tar`
- `rm`: deletes image

### Basic docker volume commands:

- `create` : creates a volume
- `inspect`: inspects it
- `ls`: lists volumes
- `prune`: removes unused local volumes
- `rm` : removes one or more volumes

## Check installation works

- `docker container run hello-world`
- `docker container run ubuntu echo hello`

## Useful commands

### Create an interactive container

- `-t` : allocates pseudo TTY
- `-i` : keep STDIN open
  => `docker container run -ti ubuntu bash`
  (bash is the default if it is not specified)
- To quit, the container : `exit`

### Foreground vs background

- By default launched on foreground
- Passing the arg `-d` launches container in the background an returs its id that we can use later

### Clean up

- `docker container run --rm MY_IMAGE` : automatically cleans up the container and removes its file system on container exit

### Name Specification

- give a name to the container: `docker container run -d --name debug alpine:3.7 sleep 1000`

### Auto restart

- `docker container run --restart=on-failure lucj/api`

## Publish port

- Allows to make the process of a container accessible
- Map a port from the process to a port from the host
- static allocation : `-p HOST_PORT:CONTAINER_PORT`
- dynamic allocation of all ports : `-P`
- NOTE : confict if several containers use the same port of the host

Example : by default nginx uses port 80. If we want to access it on port 8080 in our machine, we launch `docker container run -p 8080:80 nginx`

## Bind mount

Mount a file or folder from the host machine inside the container

- `docker container run -v HOST_PATH:CONTAINER_PATH ...`
- OR `docker container run --mount type=bind,src=HOST_PATH,dst=CONTAINER_PATH ...`
- ex: `docker container run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer`
- Note: Portainer allows to supervise my docker host. It works because we bind the socket of the docker daemon and portainer can then access the api of the docker daemon via this socket

NOTE : be careful when mounting !!
In the example below, we bind the root folder of our machine to the conainer and host folder of the container:

- `docker container run -v /:/host -ti alphine`
  Then in the container : -`# rm /host/bin/sh` -`# exit`
  Back in our machine, there is no sh command anymore

To avoid this, we can use the read only flag when mounting :

- `docker container run -v /:/host:ro -ti alphine`

## Limit resources

- By default no limit in RAP, CPU, I/O

### RAM :

estep/hogit is a process whose sole purpose is eating RAM.

- To force docker to kill the process once he attains a specific level or RAM : `docker container run --memory 32m estesp/hogit`

Note : set a soft limit with `--memory-reservation 16m`

### CPU :

- use 4 cores `docker run -it --rm progrium/stress --cpu 4`
- use 0.5 cores `docker run --cpus 0.5 -it --rm progrium/stress --cpu 4`
- use only core 1 and 3 : `docker container run --cpuset-cpus 0,3 nginx`

## Ownership

- By default, user is root. root in container is same user as root in host
- Good practice : use a non root user to create a container
  - either when creating the image (done in most official images of docker hub)
  - using the `--user` option when launching it
  - or modifying it in a container process after launching it (gosu)

## Basic docker container commands (details)

### ls

- `docker container ls`: sees all active containers
  (old version : `docker ps`)
- `docker container ls -a`: sees all active and stopped containers
- `docker container ls -q`: only get short id

### inspect

- detailed view of a container : `docker inspect 6a082350011d` : we get a very detailed json of the container
- only retrieves IP : `docker inspect --format '{{ .NetworkSettings.IPAddress }}' 6a082350011d`
- only retrieves state: `docker inspect --format '{{ json .State}}' 6a082350011d`

### logs

- See logs of container
- `-f` to have a real time update
- recommendation : writee logs on standard output (not on log files inside the container)

- `docker container run -d --name ping alpine ping 8.8.8.8`
- `docker container logs -f ping`

### exec

- Allows to launch a process in an existing container
- often used with `-it` to have an interactive shell and do some debugging

Example :

- `docker container run -d --name debug alphine:3.6 sleep 10000`
- `docker container exec -ti debug sh`

Inside the container i can launch `# ps aux`
I get

```
PID  USER  TIME  COMMAND
  1  root  0:00  sleep 10000
  8  root  0:00  sh
 15  root  0:00  ps aux
```

Note : a container is only active while the PID 1 is executing (== command specified at launch). Note that this process can not be killed easily

It can work if i specify --init :

- `docker container run --init -d --name debug alpine:3.6 sleep 10000`
- then inside the container : `# kill -15 1` kills PID 1, which stops the container

### stop

Stops one or several containers (they still exist after being stopped and can be seen with `docker container ls -a`)

- `docker container stop ID`
- `docker container stop NAME`
  Example to stop two containers : `docker container stop 56b856cbb4b0f5ddbddb69c10ce733f987203d4c18209038222c3946ebc5388b debug`

To stop all running containers :

- `docker container stop $(docker container ls -q)`

### rm

Removes one or several container. Note that a container must be stopped before being deleted.

- `docker container rm ID`
- `docker container rm NAME`

With the `-f` flag, the container is force stopped and can then be deleted.

To delete all stopped containers :

- `docker container rm $(docker container ls -aq)`

To delete all containers (running and stopped):

- `docker container rm -f $(docker container ls -aq)`

### commit

Creates an image from the current container

- `docker container commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`

Note that this approach is not recommended

### a useful command

Launch a shell in a vm :
`docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh`

# Images

## File system

- An image is a file system which contains the process that will be launched in a container and all its dependencies
- The file system is composed of several layers that are shared among images.
- An image is a portable element.
- They are distributed via a Registry (ex: Docker Hub)

Components :

- Code : (Node.js, Java,...)
- Libraries : Dependencies (Module, Jar, Gem)
- environment : Runtime (Node, JRE, Ruby)
- Binaries and system libraries : OS (Ubuntu, CentOS, Alpine)

The building of the image file system is in the opposite order of the above listed components. The file system of the image is the union of all these layers.
Example of layer per component below :

```
OS ----------->   | /bin         Runtime --> | /usr/bin/node
                  | /dev                     | /usr/bin/npm
                  | /usr                     | /usr/lib/node_modules/npm
                  | /var


Dependencies -->  | /app/node_modules/express      Code --> | /app/*.js
                  | /app/node_modules/npm

```

## Union filesystem

- An image = graph of Read only layers
- We use a storage / graph driver to:
  - Unify layers in a unique filesystem
  - add a layer in read-write linked to the container
  - This layers allows the container to modify the file system without affecting the layers of the image
- Different storage drivers are used depending on the situation
- The layers are stored in /var/lib/docker in the host machine (default install)

Note : for wsl, in the windows explorer go to : `\\wsl$\docker-desktop-data\version-pack-data\community\docker`

## Copy on write

- Mechanism that occurs when a container must modify a file in an underlying layer
- At first, the file is copied from the image read only layer to the read-write layer of the container
- Once the file is in the read/write layer, the modification can be made and be persistent

## Dockerfile

- Text file containing instructions to build the image file system
- Standard flow
  - Specification of image base
  - Adding dependencies
  - Add application code
  - define launch command

Example :

```dockerfile
# base image
FROM node:8.11.1-alpine

# copy dependencies
COPY package.json /app/package.json

# Install / compile dependencies
RUN cd /app && npm install

# Copy application code
COPY . /app/

# Expose HTTP port
EXPOSE 80

# Go to working directory
WORKDIR /app

# Command to execute on container start
CMD ["npm", "start"]
```

### FROM

- defines base image
- different possibilities (OS, server, execution env)
- `scratch` to build a minimal image which does not contain anything

I can search the list of images here: https://hub.docker.com/search?type=image

### ENV

- defines env var
- can be used inside the build
- are available in the container environment

### COPY / ADD

- Copies files and folders in the image file system
- Makes a new layer
- `--chown` to specify ownership
- `ADD` allows to specify a URL for source or unpack a tar.gz file
- `COPY` is to be privileged because we have more control on how the copy is made

### RUN

- Executes a commad in a new layer
- 2 formats:
  - Shell : launched in a shell with "/bin/sh -c" by default
  - Exec : not launched in a shell

```dockerfile
# Shell cmd
RUN apt-get update -y && apt-get install

# Not shell
RUN ["/bin/bash", "-c", "echo hello"]
```

### EXPOSE

- Informs of ports used by the app
- can be modified at container launch with `-p`

The `EXPOSE` instruction does not actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs the container, about which ports are intended to be published. To actually publish the port when running the container, use the -p flag on docker run to publish and map one or more ports, or the -P flag to publish all exposed ports and map them to high-order ports.

### VOLUME

- defines a directory whose data are decoupled from the container lifecycle
- manages data outside of the union file-system
- By default, creates a directory on host machine
- volume is initialised with data in the image
- ex : `VOLUME ["/data"]` : content of the data folder will be saved to a volume (won't be deleted on container removal)

### USER

- username or uid / gid used to launch the process of the container
- allows to prevent container to be launched by root (root on container is also root on host machine)
- used by instructions `RUN`, `CMD`, `ENTRYPOINT` following it
- user can be changed at application launch

### HEATHCHECK

- checks the health status of container
- ex : check health endpoint every 5 s

```dockerfile
FROM node:8.11-alpine
RUN apk update && apk add curl
HEALTHCHECK --interval=5x --timeout=3s --retries=3 CMD curl -f http://localhost:8000/health || exit 1
...
```

A healthcheck can allow an orchestrator such as swarm to restart a container when needed

### ENTRYPOINT / CMD

- Defines the cmd executed by the container
- `ENTRYPOINT`: binary of the application
- `CMD`: defaults args
- `ENTRYPOINT` and `CMD` are concatenated
- 2 possible formats:
  - shell, ex : `/bin/ping/localhost`
  - exec, ex : `["ping", "localhost"]`

Notes :

- shell format (ex: `ENTRYPOINT /usr/bin/node index.js`) : a command specified in this format will be executed via a shell in present in the image.
- exec format (ex: `CMD ["node", "index.js"]`) : a command in this format does not require a shell in the image to work (more robust)

ENTRYPOINT can be overwritten with --entrypoint :

- `docker container run --entrypoint /bin/sh alpine`
  Note that passing multiple args is difficult

To overwrite CMD we pass the command after the image name:

- `docker container run alpine echo "Hello"`

## Build an image

- `docker image build [OPTIONS] PATH | URL`

Common options:

- `-f` : specifies file to use (Dockerfile by default)
- `--tag / -t` : specifies image name (`[registry/]user/repository:tag`)
- `--label` : adds metadata

## List images

- lists all images: `docker image ls`
- lists all ping images: `docker image ls ping`

## Multistage build

- 1st step : creates a base image which contains libraries
- 2nd step : build production image from first image by only keeping relevant assets

This can decrease the image size

```dockerfile
# Monostage build : image size =  987MB
FROM openjdk:10
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
CMD ["java", "Main"]
```

```dockerfile
# Multistage build : image size = 295MB
FROM openjdk:10 as build
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java

FROM openjdk:10-jre-slim
COPY --from=build /usr/src/myapp/Main.class /usr/src/myapp/
WORKDIR /usr/src/myapp
CMD ["java", "Main"]
```

## Cache

Mechanism of reuse of created layers

- increase image creation speed
- ex: avoid recompiling dependencies when correcting a typo

Cache invalidation : Forces creation of a new image

- Modification of an instruction in the Dockerfile (ex: value of env var)
- Modification of copied files in the image (ADD/COPY instructions)
- If an instruction invalidates the cache, all the subsequent instructions won't use the cache

When the image is built, you can see `=> CACHED` or `---> Using Cache`

In the example below, each time i modify the source code, i have to run npm install again because cache is invalidated on line 2:

```dockerfile
FROM node:10.13-0:alpine
COPY . /app/
RUN cd /app && npm i
EXPOSE 8080
WORKDIR /app
CMD ["npm", "start"]
```

To use cache to its fullest:

```dockerfile
FROM node:10.13-0:alpine
COPY package.json /app/package.json
RUN cd /app && npm i
# cache will be invalidated at next step if i modify the source code
COPY . /app
...
```

## Build context

- When building an image, the content is packaged in a .tar and sent to the docker daemon.
- a .dockerignore file can allow to easily exclude some files/folders

The build context == all the files that go into the image

Example of .dockerignore

```dockerignore
.git
node_modules
```

## Basic docker image commands (details)

### pull

- Downloads an image from a registry (Docker Hub by default)
- Each image layer is downloaded
- Note that image is automatically downloaded when we create a container with an image that is not available locally
- name format : USER/IMAGE:VERSION (default version is latest)
- ex: `docker pull image alpine`

### push

- uploads an image in a registry (docker hub by default)
- Once uploaded, image can be used by all authorized users
- Login is necessary : `docker login [OPTIONS] [SERVER]`

### inspect

- ex :`docker image inspect`
- with format : `docker image inspect -f '{{ .Metadata }}' alpine`

### history

- shows how the image was created
- shows the history of layers
- ex : `docker image history portainer/portainer:latest`

### ls

- shows images created or downloaded
- some of them are in the dangling state (not referenced anymore, can be suppressed)
- Shows all images : `docker image ls`
- Shows all images and temporary images : `docker image ls -a`
- Shows only id : `docker image ls -q`
- filter by name : `docker image ls node`
- filter by dangling state: `docker image ls --filter dangling=true`
- delete dangling images : `docker image prune`

### save / load

- exports or load an image from a tar file
- ex : `docker save -o alpine.tar alpine` (`-o` is the same as `--output`)
- ex: `docker load < alpine.tar`
- `tar -xvf apline.tar` shows all layers : each of them contains
  - an archive (layer.tar)
  - a json file
  - a VERSION file

### rm

- removes an image and its associated layers
- ex : `docker image rm ubuntu`
- deletes all images : `docker image rm $(docker image ls -q)`

# Registry

## How to use

- A registry is a library of docker images (place where all the images are saved)
- Is is the "ship" part of the "Build Ship Run" workflow

- get image with `docker image pull`
- push image with `docker image push`

## Providers:

Docker

- Docker Hub : https://hub.docker.com (official registry)
- Docker Registry : open source solution
- Docker Trusted Registry:
  - commercial solution
  - available with Docker-EE standard/advanced

Others

- Amazon EC2 container registry
- Google Container Registry
- Quay.io (CoreOS)
- GitLab container registry

## Docker Hub

- Different images :
  - official (https://hub.docker.com/expore)
  - public / private : maintained by people/orgs
- Integration with Github / Bitbucket
- Integration in a CI/CD pipeline
- Different version of docker (CE/EE)
- Possibility to use several plugins

Docker hub allows to analyse vulnerabilities on the images.

Possibility to create a repo. The image name is `user/repository:version`

To upload a local image to docker hub, we must add the correct prefix.

- `docker image tag pong:1.3 gab24/pong:1.3`
- then login : `docker login`
- then push : `docker image push gab24/pong:1.3`

We can then see the image in the hub

## Docker Open source registry

By default, a docker daemon can only communicate with a secure connection
Exceptions:

- localhost
- pass `--insecure-registry` as an option (not recommended)
- to push an image, the image name shoud be `HOST:PORT/NAME:PORT[:VERSION]`

Example :

- launch registry in a container : `docker container run -d -p 5000:5000 registry:2.6.2`
- list images in the registry : `curl localhost:5000/v2/_catalog`. We get `{"repositories":[]}`
- pull an image from docker hub: `docker image pull nginx:1.14`
- tag it : `docker image tag nginx:1.14 localhost:5000/nginx:1.14`
- push it : `docker image push localhost:5000/nginx:1.14`
- list images in registry: `curl localhost:5000/v2/_catalog`: we get `{"repositories": ["nginx"]}`

# Storage

## Containers and data persistence

- In a container, changes are saved to the layer in read/write
- These changes are lost if the container is destroyed.
- To be decoupled from container lifecycle, data must be handled outside of the filesystem

## Volumes

- files outside the union filesystem
- created in different ways
- `VOLUME` in the Dockerfile
- options `-v` / `--mount` when creating a container
- via cmd `docker volume create`

Use cases :

- data persistence in a database
- logs persistance outside of container

Example :

- create a volume named db-data : `docker volume create --name db-data`
- launch mongo db and persist data in db-data volume : `docker container run -d --name db -v db-data:/data/db mongo:4.0`

To find the path of the volumes : `docker volume inspect -f '{{ .Mountpoint }}' db-data`
Note that in wsl, i can not find the volume in `/var/lib/docker/volumes/db-data/_data`.
Instead, in the file explorer, i go to `\\wsl$\docker-desktop-data\version-pack-data\community\docker`

## Volume drivers

- Volume drivers allow creation of volume on different types of storage
- By default : "local":
  - storage on host machine
  - in `/var/lib/docker/volume/ID` (or `\\wsl$\docker-desktop-data\version-pack-data\community\docker\volumes/ID`)
- Additional drivers can be installed via plugins
  - can allow to use external storage (Ceph, AWS, GCE, Azure)

## Plugins

Note : plugins can be seen at https://hub.docker.com/search?type=plugin

### sshfs

Allows to store on a file system via ssh :

- https://github.com/vieux/docker-volume-sshfs
- example below is not production ready

Using the plugin:

- Install the plugin : `docker plugin install vieux/sshfs`

- Check the plugin was enabled: `docker plugin ls -f enabled=true` (`-f` for `--filter`)

- create a volume in the remote: `docker volume create -d vieux/sshfs -o sshcmd=<user>@<sshhost>:<sshpath> -o password=<password> <volumename>`
- launch a container do map the volume : `docker container run -it -v <volumename>:<pathincontainer> alpine:3.8`

In the container, if i create a file in `<pathincontainer>`, then it will be visible in the remote

# Docker machine (DEPRECATED)

- Installs docker (client and daemon) either on physical machine or VM
- allows local client to communicate with distant daemon
- certificates are delivered at creation

Use :

- Local:
  - Oracle Virtualbox
  - VMWare
- Cloud:
  - DigitalOcean
  - AWS
  - Azure
  - Google Compute Engine
- Generique : used to create a machine on a vm or a physical machine

On a local machine:

- Client (docker) communicates with local daemon (dockerd) via socket /var/run/docker.sock

We can also use docker machine with the `-d (or --driver)` option to create docker hosts (docker and dockerd) on a local machine or on a cloud provider :

- `docker-machine create -d amazonec2 ...`
- `docker-machine create -d google ...`
- `docker-machine create -d virtualbox ...`

Once we created a machine, it is possible to configure the client to address the distant daemon via a tcp socket (and not the local). To do so, we have to set some env variable. It can be done in a single command: `eval $(docker-machine env <machinename>)`

..........

# Docker compose

- Good for a microservice architecture !
- docker-compose.yml : defines all the services used in an app
- docker-compose binary: allows to launch the app in docker-compose.yml

## docker-compose.yml

https://docs.docker.com/compose/

Defines all elements of an app:

- services
- volumes
- networks
- secrets (only used by swarm)
- configs (only used by swarm)

Example :

```yml
version: '3.7' # compose file format
volumes:
  data: # volume data will be used by a service. By default we use a local driver
networks: # definition of networks to isolate services
  frontend:
  backend:
services: # definition of 3 services that makes the app
  web:
    image: org/web:2.3 # image used by the web service
    networks:
      - frontend # only accesse frontend network
    ports:
      - 80:80 # port mapping
  api:
    image: org/api:1.2
    networks: # accesses both networks
      - backend
      - frontend
  db:
    image: mongo:4.0
    volumes:
      - data:/data/db # mounts data volume to /data/db
    networks:
      - backend
# web and db can not commmunicate directly because they dont access the same network
```

Options for a service in docker compose :

- image used : `image: <imagename>`
- nb of replicas == nb of identical container launched for the service: `replicas <nb>`
- ports published : `ports:`
- Health check: `healthcheck:`
- restart strategy: `restart:<strategy>`
- redefines container command : `command: <command>`
- swarm only: deployment constraints
- swarm only: updates configs
- swarm only: secrets used
- swarm only: config used

Note : build allows to specify a path to a dockerfile.

```yml
services:
  web:
    build: ./project # finds Dockerfile in project dir
    ...
  db:
    build:
      context: ./project/db # goes to project/db folder
      dockerfile: Dockerfile-alternate # uses alternate dockerfile
```

## docker-compose binary

- handles lifecycle of an application with Compose format
- independant of the docker daemon but often shipped together
- written in python

NOTE : `docker compose` is a rewrite of `docker-compose` in Go

Main commands:

- `docker-compose up`: creates application
  - use `-d` to detach
- `docker-compose down`: removes application
  - use `-v` to remove volumes
- `docker-compose start <service>`: starts an existing container
- `docker-compose stop <service>`: stops an existing container without removing it
- `docker-compose build`: builds images of services (if build instruction is used)
- `docker-compose pull`: Pulls images for services defined in a Compose file, but does not start the containers.
- `docker-compose logs`: view output of containers
- `docker-compose scale`: modifies number of container for a service
- `docker-compose ps`: lists containers of the app

## service discovery

- use of docker daemon DNS to resolve services
- a service communicates with another via his name

ex : in a docker-compose.yml

```yml
services:
  db: ...
```

In a node.js code file:

```javascript
// Mongodb connection string
url = 'mongodb://db/todos'; // db is the name of the service
```

Note that docker compose allows to define dependencies between services but does not allow to know that a service is available before launching a service that depends on it.

For example, if i have a web service that depends on a postgres service, in the dockerfile used to build the web service, i can add the following :

```dockerfile
# DockerFile
COPY ./entrypoint.sh .
RUN chmod +x /workdir/entrypoint.sh

# run entrypoint.sh
ENTRYPOINT ["/workdir/entrypoint.sh"]
```

where entrypoint.sh is:

```sh
#!/bin/sh
echo "Waiting for postgres..."
while ! nc -z db 5432; do
  sleep 0.1
done
echo "PostgreSQL started"
exec "$@"
```

Notes:

- `nc -z`: Only scan for listening daemons, without sending any data to them.
- `exec "$@"` basically takes any command line arguments passed to `entrypoint.sh` and execs them as a command

## Voting app

- https://github.com/dockersamples/example-voting-app
- Open source application maintained by docker
- 5 services:
  - 3 languages: Node.js / python / .NET
  - Two databases: Redis / Postgres

```
voting-app (python) -> redis -> worker (.NET) -> postgres -> result-app (Node.jss)
```

A user votes via web interface -> vote is stored in redis -> the worker gets the vote in redis and saves it in postgres -> users can see the result on another web interface

- The repo contains a lot of different docker-compose files : we will look in detail at docker-compose-simple.yml (used for development)
  => `docker-compose -f docker-compose-simple.yml up -d`

# Docker Swarm

Since v1.12:

- orchestration is integrated to docker daemon
- easy to setup
- secure by default
- usable in dev environment

## Swarm mode

Typical workflow:

- define an app with docker compose format
- deploy it in the cluster

Swarm mode : primitives

- Node = machine member of a swarm cluster
  - Managers: manage the cluster
  - Workers: run the containers of the app
- Service = allows to specify the way we want to launch the containers of the app
- Stack = group of service
- Secret = sensitive data
- Config = config of the app

Using a swarm is easy:

- create a swarm : `docker swarm init`
- add a node: `docker swarm joing`

## Raft: consensus in swarm mode

- Consensus involves multiple servers agreeing on values
- Swarm uses raft. An animation explaining it can be found here: http://thesecretlivesofdata.com/raft/

## Node

- Machine which is part of the swarm and where docker daemon is running
- Types:
  - Manager:
    - schedules and orchestrate containers
    - manages cluster state
    - uses RAFT consensus algorithm
    - also a worker by default
  - Worker:
    - executes containers
- Several possible states:
  - Active: the scheduler can assign tasks to the node
  - Pause:
    - the scheduler can not assign tasks to the nod
    - currently running tasks remain unchanged
  - Drain:
    - the scheduler can not assign tasks to the nod
    - the running tasks on the node are stopped and launched on other nodes
- access node api : `docker node --help`
