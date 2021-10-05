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

### docker run argument:

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

## Check installation works

- `docker container run hello-world`
- `docker container run ubuntu echo hello`

## Useful commands

### Create an interactive container

- `-t` : allocates pseudo TTY
- `-i` : keep STDIN open
  => `docker container run -ti ubuntu bash`
  (bash is the default if it is not specified)
  To quit, the container : `exit`

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
  Portainer allows to supervise my docker host. It works because we bind the socket of the docker daemon and portainer can then access the api of the docker daemon via this socket

-NOTE : be careful when mounting !!
in the example below, we bind the root folder of our machine to the conainer and host folder of the container:

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

`docker container ls`: sees all active containers
(old version : `docker ps`)
`docker container ls -a`: sees all active and stopped containers
`docker container ls -q`: only get short id

### inspect

- detailed view of a container : `docker inspect 6a082350011d` : we get a very detailed json of the container
- only retrieves IP : `docker inspect --format '{{ .NetworkSettings.IPAddress }}' 6a082350011d`
  -only retrieves state: `docker inspect --format '{{ json .State}}' 6a082350011d`

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

Note : for wsl, in the windows explorer go to : `\\wsl$\docker-desktop-data\version-pack-data\community\docker\volumes`

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