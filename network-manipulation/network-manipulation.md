# Docker Network Basics

A network in Docker is another logical object like a container and image.   
To list out the networks in the system:
```
docker network ls
```
By default, Docker has five networking drivers. They are as follows:
- ```bridge``` - The default networking driver in Docker. This can be used when multiple containers are running in standard mode and need to communicate with each other.
- ```host``` - Removes the network isolation completely. Any container running under a host network is basically attached to the network of the host system.
- ```none``` - This driver disables networking for containers altogether. I haven't found any use-case for this yet.
- ```overlay``` - This is used for connecting multiple Docker daemons across computers and is out of the scope of this book.
- ```macvlan``` - Allows assignment of MAC addresses to containers, making them function like physical devices in a network.

---
### Create a User-Defined Bridge in Docker
---
Docker comes with a default bridge network named bridge. Any container you run will be automatically attached to this bridge network:
```
docker container run --rm --detach --name hello-dock --publish 8080:80 < container name >

docker network inspect --format='{{range .Containers}}{{.Name}}{{end}}' bridge
```
Containers attached to the default bridge network can communicate with each others using IP addresses which is not recommended (IP can change).   
A user-defined bridge, however, has some extra features over the default one:

- User-defined bridges provide automatic DNS resolution between containers: This means containers attached to the same network can communicate with each others using the container name. So if you have two containers named notes-api and notes-db the API container will be able to connect to the database container using the notes-db name.   
  
- User-defined bridges provide better isolation: All containers are attached to the default bridge network by default which can cause conflicts among them. Attaching containers to a user-defined bridge can ensure better isolation.
  
- Containers can be attached and detached from user-defined networks on the fly: During a container’s lifetime, you can connect or disconnect it from user-defined networks on the fly. To remove a container from the default bridge network, you need to stop the container and recreate it with different network options.   
  
A network can be created using the network create command. 
```
docker network create <network name>

docker network create skynet
```

---
### Attach a Container to a Network in Docker
---
There are mostly two ways of attaching a container to a network.

```
docker network connect <network identifier> <container identifier>

docker network connect skynet hello-dock
```
The second way of attaching a container to a network is by using the ```--network``` option for the container run or container create commands.
```
docker container run --network skynet --rm --name alpine-box -it alpine sh
```
Here, containers are under the same user-defined bridge network and automatic DNS resolution is working.

Keep in mind, though, that in order for the automatic DNS resolution to work you must assign custom names to the containers. Using the randomly generated name will not work.

---
### Detach Containers from a Network in Docker
---
```
docker network disconnect <network identifier> <container identifier>

docker network disconnect skynet hello-dock
```

---
### Remove networks from Docker
---
Just like the other logical objects in Docker, networks can be removed using the network rm command:
```
docker network rm <network identifier>

docker network rm skynet
```
