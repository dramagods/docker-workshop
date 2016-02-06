# [![harbur.io](https://en.gravatar.com/userimage/10968596/06879c44248462a1bac025dd999fe704.png?size=64)](http://harbur.io) Docker Workshop [![Gitter chat](https://badges.gitter.im/harbur/docker-workshop.png)](https://gitter.im/harbur/docker-workshop)

The Workshop is separated in three sections

* [CLI Basics](#cli-basics)
* [Dockerfile basics](#dockerfile-basics)
* [Docker Patterns](#docker-patterns)
* [Docker Compose](#docker-compose)

Preparations:

* Install Docker
* Clone this repo: `git clone https://github.com/dramagods/docker-workshop` (Some code examples require files located here)
* Warm-up the images:

```
docker pull tomcat:8
docker pull sebp/elk
docker pull redis
docker pull java:7-jre
docker pull ubuntu
docker pull nginx
```

Assumptions:

* During workshop the following ports are used `4000-4010`, 5000, 5601, 8080 and 9200. If they are not available on your machine, adjust the CLI commands accordingly.

# CLI Basics

### Version

Check you have latest version of docker installed:

```
docker version
```

* If you don't have docker installed, check [here](https://docs.docker.com/installation/#installation)
* If you're not on the latest version, it will prompt you to update
* If you're not on docker group you might need to prefix commands with `sudo`. See [here](http://docs.docker.com/installation/ubuntulinux/#giving-non-root-access) for details about it.

### Commands

Check the available docker commands

```
docker
```

* Whenever you don't remember a command, just type docker
* For more info, type `docker help COMMAND` (e.g. `docker help run`)

### RUN a "Hello World" container

```
docker run tomcat:8 echo "Hello World"
```

* If the Image is not cached, it pulls it automatically
* It prints `Hello World` and exits

###  RUN a while loop in a deamonized container
```
docker run -d ubuntu /bin/bash -c "while true; do echo hello; sleep 1; done"
```

* It prints hello endless

### RUN an interactive Container

```
docker run -it tomcat:8 bash
  cat /etc/os-release
```

* **-i**: Keep stdin open even if not attached
* **-t**: Allocate a pseudo-tty

### RUN a Container with pipeline

```
cat /etc/resolv.conf | docker run -i tomcat:8 wc -l
```

### SEARCH a Container

```
docker search -s 10 tomcat
```

* **-s**: Only displays with at least x stars

### RUN a Container and expose a Port

```
docker run -d -p 8080:8080 tomcat:8
google-chrome localhost:8080 (if you're using linux)
google-chrome $(boot2docker ip):8080 (if you're using mac/win)
```

* **-d**: Detached mode: Run container in the background, print new container id
* **-p**: Publish a container's port to the host (format: *hostPort:containerPort*)
* For more info about the container, see [nginx](https://registry.hub.docker.com/_/nginx/)

### RUN a Container with a Volume
Example 1:
```
docker run -d -p 4001:80 -v $(pwd)/hello-world/site/:/usr/share/nginx/html:ro nginx
google-chrome localhost:4001 (if you're using linux)
google-chrome $(boot2docker ip):4001 (if you're using mac/win)
```

Example 2:
```
docker run -d -p 8080:8080 -v ${PWD}/tomcat/conf:/usr/local/tomcat/conf tomcat:8
google-chrome localhost:8080 (if you're using linux)
google-chrome $(boot2docker ip):8080 (if you're using mac/win)

docker exec -ti $(docker ps -lq) bash

docker inspect $(docker ps -lq)
```

* **-v**: Bind mount a volume (e.g., from the host: -v /host:/container, from docker: -v /container)
* The volume is **linked** inside the container. Any external changes are visible directly inside the container.
* This example breaks the immutability of the container, good for debuging, not recommended for production (Volumes should be used for data, not code)
* **exec**: Run a command inside the container
* **inspect**: Provides detailed information of a container

# Dockerfile Basics

### BUILD a Tomcat Container with Ant installed

Create a Tomcat Container with ant manually:

```
docker run -it --name ant tomcat:8 bash
  apt-get update
  apt-get -y install ant
  ant -version
  exit
docker commit ant docker-tomcat-ant
docker rm ant
docker run --rm -it docker-tomcat-ant ant -version
docker rmi docker-tomcat-ant
```

* **--name**: Assign a name to the container
* **commit**: Create a new image from a container's changes
* **rm**: Remove one or more containers
* **rmi**: Remove one or more images
* **--rm**: Automatically remove the container when it exits

Create a Tomcat Container with Ant installed using Dockerfile:

```
cd docker-tomcat-ant
docker build -t docker-tomcat-ant .
docker run -it docker-tomcat-ant ant -version
```

* **build**: Build an image from a Dockerfile

[docker-tomcat-ant/Dockerfile](docker-tomcat-ant/Dockerfile)
```
FROM tomcat:8

MAINTAINER Alfonso Gonzalez <alfonso@offsidegaming.com>

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update
RUN apt-get -y install ant
```

* The **FROM** instruction sets the Base Image for subsequent instructions
* The **MAINTAINER** instruction sets person/s who maintain the Dockerfile
* The **RUN** instruction will execute any commands in a new layer on top of the current image and commit the results
* The **ENV** instruction sets the environment variable <key> to the value <value>

### BUILD an Apache Tomcat Server Container

Create an Apache Tomcat Server Container with Dockerfile:

```
cd docker-tomcat-build
docker build -t docker-tomcat-build .
docker run -d -p 8080:8080 docker-tomcat-build
google-chrome localhost:8080 (if you're using linux)
google-chrome $(boot2docker ip):8080 (if you're using mac/win)
```

[docker-tomcat-build/Dockerfile](docker-tomcat-build/Dockerfile) Taken from Official Docker Image https://github.com/docker-library/tomcat/blob/2f17559d7a1c62dbc17b63750b31a0beea205120/8-jre7/Dockerfile
```
FROM java:7-jre

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME

# see https://www.apache.org/dist/tomcat/tomcat-8/KEYS
RUN gpg --keyserver pool.sks-keyservers.net --recv-keys \
  05AB33110949707C93A279E3D3EFE6B686867BA6 \
  07E48665A34DCAFAE522E5E6266191C37C037D42 \
  47309207D818FFD8DCD3F83F1931D684307A10A5 \
  541FBE7D8F78B25E055DDEE13C370389288584E7 \
  61B832AC2F1C5A90F0F9B00A1C506407564C17A3 \
  79F7026C690BAA50B92CD8B66A3AD3F4F22C4FED \
  9BA44C2621385CB966EBA586F72C284D731FABEE \
  A27677289986DB50844682F8ACB77FC2E86E29AC \
  A9C5DF4D22E99998D9875A5110C01C5A2F6059E7 \
  DCFD35E0BF8CA7344752DE8B6FB21E8933C60243 \
  F3A04C595DB5B6A5F1ECA43E3B7BBB100D811BBE \
  F7DA48BB64BCB84ECBA7EE6935CD23C10D498E23

ENV TOMCAT_MAJOR 8
ENV TOMCAT_VERSION 8.0.24
ENV TOMCAT_TGZ_URL https://www.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz

RUN set -x \
  && curl -fSL "$TOMCAT_TGZ_URL" -o tomcat.tar.gz \
  && curl -fSL "$TOMCAT_TGZ_URL.asc" -o tomcat.tar.gz.asc \
  && gpg --verify tomcat.tar.gz.asc \
  && tar -xvf tomcat.tar.gz --strip-components=1 \
  && rm bin/*.bat \
  && rm tomcat.tar.gz*

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

* The **WORKDIR** instructions change or makes the following directory the current working directory
* The **EXPOSE** instructions informs Docker that the container will listen on the specified network ports at runtime
* The **CMD** instruction sets the command to be executed when running the image

### BUILD a Hello Tomcat  Image

```
cd docker-tomcat-build
docker build -t docker-tomcat-build .
docker run -d --name hello-tomcat -P docker-tomcat-build
google-chrome $(docker port hello-tomcat 8080)
```

* **-P**: Publish all exposed ports to the host interfaces
* **port**: Lookup the public-facing port that is NAT-ed to PRIVATE_PORT

[hello-world/Dockerfile](hello-world/Dockerfile)


```
cd hello-world
cat Dockerfile
  FROM nginx:1.9.3
  ADD site /usr/share/nginx/html
```

Build hello-world image

```
docker build -t hello-world .
```

* The **ADD** instruction will copy new files from <src> and add them to the container's filesystem at path <dest>. It allows a URL in <src>. If the <src> parameter of ADD is an archive in a recognised compression format, it will be unpacked.
* The **COPY** instruction will copy new files from <src> and add them to the container's filesystem at path <dest>

### PUSH Image to a Registry

```
REGISTRY=docker.offsidegaming.com:5000
docker tag hello-world $REGISTRY/offsidegaming/hello-world
docker push $REGISTRY/offsidegaming/hello-world
```

* **tag**: Tag an image into a repository
* **push**: Push an image or a repository to a Docker registry server

### PULL Image from a Repository

```
docker pull $REGISTRY/offsidegaming/hello-world
docker run -d -P --name=registry-hello $REGISTRY/offsidegaming/hello-world
google-chrome $(docker port registry-hello 80)
```

* **pull**: Pull an image or a repository from a Docker registry server

# Docker Patterns

## [Linking Containers Together](https://docs.docker.com/userguide/dockerlinks/)

```
docker run -d --name redis redis
docker run -it --rm --link redis:server redis bash
  env
  cat /etc/hosts
  
docker run -it --rm --link redis:server redis bash -c 'redis-cli -h $SERVER_PORT_6379_TCP_ADDR'
  set hello world
  get hello
```

## [Data Volume Pattern](http://docs.docker.com/userguide/dockervolumes/)

```
docker run -d -p 4004:80 --name web -v /usr/share/nginx/html nginx
google-chrome localhost:4004
docker run --rm -it --volumes-from web ubuntu bash
  vi /usr/share/nginx/html/index.html
```


# Docker compose (https://docs.docker.com/compose/)

Compose is a tool for defining and running multi-container applications with Docker. With Compose, you define a multi-container application in a single file, then spin your application up in a single command which does everything that needs to be done to get it running.

Compose is great for development environments, staging servers, and CI. We don’t recommend that you use it in production yet.

Using Compose is basically a three-step process.

Define your app’s environment with a Dockerfile so it can be reproduced anywhere.
Define the services that make up your app in docker-compose.yml so they can be run together in an isolated environment:
Lastly, run docker-compose up and Compose will start and run your entire app.
A docker-compose.yml looks like this:

```
calendarapp:
  build: .
  environment:
    - ENV=production
  ports:
    - "8080:8080"
  volumes:
    - tomcat/conf:/usr/local/tomcat/conf
  links:
    - elk
  dns:
    - 10.1.1.121
elk:
  image: sebp/elk
  links:
    - redis
  ports:
    - "5601:5601"
    - "9200:9200"
    - "5000:5000"
  volumes:
    - ./logstash:/etc/logstash/conf.d/
redis:
  image: redis
```

### Docker compose installation (https://docs.docker.com/compose/install)

# CLI Basics

### Version

Check you have latest version of docker installed:

```
docker-compose --version
```

* If you don't have docker compose installed, check [here](https://docs.docker.com/compose/install)

### Commands

Check the available docker-compose commands

```
docker-compose
```

* Whenever you don't remember a command, just type docker
* For more info, type `docker-compose help COMMAND` (e.g. `docker-compose help run`)

```
cd docker-tomcat
docker-compose up
C^ + C
docker-compose rm
docker-compose up -d
docker-compose ps
docker-compose run calendarapp env
docker-compose logs
docker-compose stop
docker-compose rm

```

* **up**: it will download or build the images and start containers in a proper order. Use -d to run it in daemon.
* **rm**: Removes all containers
* **ps**: Shows containers running.
* **logs**: Shows containers console logs.
* **stop**: Stop containers

# Helpers commands

Cleanup Stopped Containers:

```
docker rm $(docker ps -q -a)
```

Cleanup Untagged Images:

```
docker rmi $(docker images -q -f dangling=true)
```

# Credits

This workshop was prepared by Alfonso Gonzalez based on harbur docker workshop[harbur.io](http://harbur.io), under MIT License. Feel free to fork and improve.
