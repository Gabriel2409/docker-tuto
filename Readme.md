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

### a useful command

Launch a shell in a vm :
`docker run -it --privileged --pid=host debian nsenter -t 1 -m -u -n -i sh`
