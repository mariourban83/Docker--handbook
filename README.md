# Docker Introduction

According to IBM,

Containerization involves encapsulating or packaging up software code and all its dependencies so that it can run uniformly and consistently on any infrastructure.

Example of minimal containerization with Docker:
```
docker run hello-world
```
To have a look at all the containers that are currently running or have run in the past:
```
docker ps -a
```
Docker Architecture and three very fundamental concepts of containerization in general:
- Container

    A container is an abstraction at the application layer that packages code and dependencies together. Instead of virtualizing the entire physical machine, containers virtualize the host operating system only.

- Image

    Images are multi-layered read-only files carrying your application in a desired state inside them.
    Images can be exchanged through registries.

- Registry

    An image registry is a centralized place where you can upload your images and can also download images created by others. Docker Hub is the default public registry for Docker.

---
## Docker Engine
---

The engine consists of three major components:

### Docker Daemon:
The daemon (dockerd) is a process that keeps running in the background and waits for commands from the client. The daemon is capable of managing various Docker objects.

### Docker Client:
The client  (docker) is a command-line interface program mostly responsible for transporting commands issued by users.

### REST API:
The REST API acts as a bridge between the daemon and the client. Any command issued using the client passes through the API to finally reach the daemon.



![Docker Explained](assets/docker1.svg "Docker Explained")

---
## Docker Container Manipulation 
---
Container manipulation is one of the most common task so a proper understanding of the various commands is crucial.

The generic syntax:
```
docker <object> <command> <options>
```
* object indicates the type of Docker object you'll be manipulating. This can be a container, image, network or volume object.
* command indicates the task to be carried out by the daemon, that is the run command.
* options can be any valid parameter that can override the default behavior of the command, like the --publish option for port mapping.

---
### Publishing a Port
---
Containers are isolated environments. Your host system doesn't know anything about what's going on inside a container. Hence, applications running inside a container remain inaccessible from the outside. To allow access from outside of a container, you must publish the appropriate port inside the container to a port on your local network. The common syntax for the --publish or -p option is as follows:
```
--publish <host port>:<container port>

--publish 8080:80  (request sent to port 8080 of host system will be forwarded to port 80 inside the container‌)
```
---
### Using Detached Mode
---
To keep the container running even if terminal is closed:
```
docker container run --detach --publish 8080:80 registryName/imageName
```
---
### Listing containers
---
To list running containers only:
```
docker container ls
```
To list all containers, including ones running in the past :
```
docker container ls --all
```
---
### Naming and renaming containers
---
By default, every container has two identifiers. They are as follows:

* CONTAINER ID - a random 64 character-long string.
* NAME - combination of two random words, joined with an underscore.

To give container a name:
```
docker container run --detach --publish 8888:80 --name < new Name > < registryName/oldName >
```
To rename old containers:
```
docker container rename <container identifier> <new name>
```
---
### Start, stop, restart, kill containers
---
To stop container running in the foreground:
```
ctrl + C
```
for background running container:
```
docker container stop
                 kill
                 restart < container name >
```
To start container:
```
docker container start < container id or name >
```
---
### Create and remove containers
---
To only create container without starting it and start it later:
```
docker container create --publish 8080:80 < registry/name >

docker container start < container name >
```

Containers that have been stopped or killed remain in the system. These dangling containers can take up space or can conflict with newer containers.

To remove stopped, killed, or not used containers:
```
docker container rm < container id or name> < container id or name> < container id or name> ...
```
---
### Run Container in Interactive mode
---
```
docker container run --rm -it ubuntu
```
With -it flag, container will start and shell from the inside of the container will open.   
The --rm flag will remove container when stopped or killed.

---
### Execute comands inside containers:
---
```
docker container run <image name> <command>
docker run alpine uname -a
docker container run --rm busybox echo -n my-secret | base64
```
Most of the images except the executable images use shell or sh as the default entry-point.     
Any valid shell command can be passed to them as arguments.   
Whatever is passed after the image name gets passed to the default entry point of the image.

---
### Working with executable images & bind mounts
---
These images are designed to behave like executable programs.   
One way to grant a container direct access to your local file system is by using bind mounts.   

When you use a bind mount, a file or directory on the host machine is mounted into a container
A bind mount lets you form a two way data binding between the content of a local file system directory (source) and another directory inside a container (destination). This way any changes made in the destination directory will take effect on the source directory and vise versa.
```
docker run -d \
  -it \
  --name devtest \
  -v "$(pwd)"/target:/app \
  nginx:latest
```
If you use -v or --volume to bind-mount a file or directory that does not yet exist on the Docker host, -v creates the endpoint for you. It is always created as a directory.
```
docker run -d \
  -it \
  --name devtest \
  --mount type=bind,source="$(pwd)"/target,target=/app \
  nginx:latest
  ```
If you use --mount to bind-mount a file or directory that does not yet exist on the Docker host, Docker does not automatically create it for you, but generates an error.

---
## Docker Image Manipulation Basics
---

Images are multi-layered self-contained files that act as the template for creating Docker containers.
To perform an image build, the daemon needs two very specific pieces of information. These are the name of the Dockerfile and the build context. 

To create Dockerfile:
```
FROM ubuntu:latest

EXPOSE 80

RUN apt-get update && \
    apt-get install nginx -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

CMD ["nginx", "-g", "daemon off;"]
```
After that:
```
docker image build .
``` 

- docker image build is the command for building the image. The daemon finds any file named Dockerfile within the context.
- The . at the end sets the context for this build. The context means the directory accessible by the daemon during the build process.

To run created container:
```
docker container run --rm --detach --name custom-nginx-packaged --publish 8080:80 < container ID >
```
---
### Tag Docker images
---
Just like containers, you can assign custom identifiers to your images instead of relying on the randomly generated ID. In case of an image, it's called tagging instead of naming. The --tag or -t option is used in such cases.
```
--tag <image repository>:<image tag>
docker image build --tag custom-nginx:packaged .
```
Example: If you want to run a container using a specific version of MySQL, like 5.7, you can execute 'docker container run mysql:5.7' where mysql is the image repository and 5.7 is the tag.   

---
### List and Remove Docker Images
---
Just like the container ls command, you can use the image ls command to list all the images in your local system:
```
docker image ls
```
Images listed can be then deleted using the image rm command. The generic syntax is as follows:
```
docker image rm <image identifier>
docker image rm custom-nginx:packaged
```
Also the 'image prune' command can be used to cleanup all un-tagged dangling images as follows:
```
docker image prune --force
```
---
### Layers of a Docker Image
---
Images are multi-layered files. To visualize the many layers of an image, you can use the image history command. The various layers of the custom-nginx:packaged image can be visualized as follows:
```
docker image history custom-nginx:packaged
```
Image comprises of many read-only layers, each recording a new set of changes to the state triggered by certain instructions. When you start a container using an image, you get a new writable layer on top of the other layers. By utilizing this concept, Docker can avoid data duplication and can use previously created layers as a cache for later builds. This results in compact, efficient images that can be used everywhere.

---
## EXAMPLE : Build NGINX from Source
---
The image creation process this time can be done in seven steps. These are as follows:

- Get a good base image for building the application, like ubuntu.
- Install necessary build dependencies on the base image.
- Copy the nginx-1.19.2.tar.gz file inside the image.
- Extract the contents of the archive and get rid of it.
- Configure the build, compile and install the program using the make tool.
- Get rid of the extracted source code.
- Run nginx executable.

Dockerfile:
```
FROM ubuntu:latest

RUN apt-get update && \
    apt-get install build-essential\ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl-dev \
                    -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

COPY nginx-1.19.2.tar.gz .

RUN tar -xvf nginx-1.19.2.tar.gz && rm nginx-1.19.2.tar.gz

RUN cd nginx-1.19.2 && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install

RUN rm -rf /nginx-1.19.2

CMD ["nginx", "-g", "daemon off;"]
```
Now to build this image:
```
docker image build --tag custom-nginx:built .
```
Updated Dockerfile with ARG and ADD
```
FROM ubuntu:latest

RUN apt-get update && \
    apt-get install build-essential\ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl-dev \
                    -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION}

RUN cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install

RUN rm -rf /${FILENAME}}

CMD ["nginx", "-g", "daemon off;"]
```
The ARG instruction allows to declare variables like in other languages. These variables or arguments can later be accessed using the ${argument name} syntax.   
Run container to test the image:
```
docker container run --rm --detach --name custom-nginx-built --publish 8080:80 custom-nginx:built
```

---
### Optimize Docker Images
---   

Remove unnecessary files after installation, keep them in single RUN command so they will not be included in any layer.   
Updated Dockerfile:
```
FROM ubuntu:latest

EXPOSE 80

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN apt-get update && \
    apt-get install build-essential \ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl-dev \
                    -y && \
    tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION} && \
    cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install && \
    cd / && rm -rfv /${FILENAME} && \
    apt-get remove build-essential \ 
                    libpcre3-dev \
                    zlib1g-dev \
                    libssl-dev \
                    -y && \
    apt-get autoremove -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

CMD ["nginx", "-g", "daemon off;"]
```
The image size has gone from being 343MB to 81.6MB. The official image is 133MB. This is a pretty optimized build, but we can go a bit further using Alpine Linux.    

---
### Alpine Linux
---   
Where the latest ubuntu image weighs at around 28MB, alpine linux is 2.8MB.  
Apart from the lightweight nature, Alpine is also secure and is a much better fit for creating containers than the other distributions.
      
Updated Dockerfile for building NGINX with Alpine Linux
```
FROM alpine:latest

EXPOSE 80

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN apk add --no-cache pcre zlib && \
    apk add --no-cache \
            --virtual .build-deps \
            build-base \ 
            pcre-dev \
            zlib-dev \
            openssl-dev && \
    tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION} && \
    cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install && \
    cd / && rm -rfv /${FILENAME} && \
    apk del .build-deps

CMD ["nginx", "-g", "daemon off;"]
```
---
### Sharing Docker Images Online
---
Login to Docker with ``` docker login ``` via cmd.   
In order to share an image online, the image has to be tagged.
```
--tag <image repository>:<image tag>

docker image build --tag mariourban83/custom-nginx:latest --file Dockerfile .
```
The image name can be anything and can not be changed once the image is uploaded. The tag can be changed whenever needed and usually reflects the version of the software or different kind of build.   

Once the image has been built, it can be uploaded by executing the following command:
```
docker image push <image repository>:<image tag>

docker image push mariourban83/custom-nginx:latest
```
Once it's done, the image should be visible in user DockerHub profile page.

---   

For Educational purposes only.   

Author : Farhan Hasin Chowdhury    

Original content website:    
```https://www.freecodecamp.org/news/the-docker-handbook/```