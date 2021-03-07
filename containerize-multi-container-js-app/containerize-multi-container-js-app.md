# Containerize a full-fledged multi-container project

 The project : simple notes-api powered by Express.js and PostgreSQL.   
 ( two containers connected using network with env vars and named volumes set )  

 ---
 ### Run the Database Server
 ---
The database server in this project is a simple PostgreSQL server and uses the official postgres image.

According to the official docs, in order to run a container with this image, you must provide the POSTGRES_PASSWORD environment variable. Apart from this one, I'll also provide a name for the default database using the POSTGRES_DB environment variable. PostgreSQL by default listens on port 5432, so you need to publish that as well. 

To run the database server you can execute the following command:
```
docker container run \
    --detach \
    --name=notes-db \
    --env POSTGRES_DB=notesdb \
    --env POSTGRES_PASSWORD=secret \
    --network=notes-api-network \
    postgres:12
```
The ```--env``` option for the container run and container create commands can be used for providing environment variables to a container.   

Databases like PostgreSQL, MongoDB, and MySQL persist their data in a directory. PostgreSQL uses the /var/lib/postgresql/data directory inside the container to persist data.

Now what if the container gets destroyed for some reason? You'll lose all your data. To solve this problem, a named volume can be used.   

---
### Work with Named Volumes in Docker
---
A named volume is very similar to an anonymous volume except that you can refer to a named volume using its name.   

Volumes are also logical objects in Docker and can be manipulated using the command-line. The ```volume create``` command can be used for creating a named volume:
```
docker volume create <volume name>

docker volume create notes-db-data
```
Now, run the container with :
```
docker container run \
    --detach \
    --volume notes-db-data:/var/lib/postgresql/data \
    --name=notes-db \
    --env POSTGRES_DB=notesdb \
    --env POSTGRES_PASSWORD=secret \
    --network=notes-api-network \
    postgres:12
```
Now the data will safely be stored inside the notes-db-data volume and can be reused in the future. A bind mount can also be used instead of a named volume here.

---
### Access Logs from a Container in Docker
---
To see logs from the container:
```
docker container logs <container identifier>
```
There is also ```--follow``` or ```-f``` option for the command which lets you attach the console to the logs output and get a continuous stream of text.

---
### Create a Network and Attach the Database Server in Docker
---
The containers have to be attached to a user-defined bridge network in order to communicate with each other using container names. To do so, create a network first:
```
docker network create notes-api-network

and 

docker network connect notes-api-network notes-db
```
to attach ```notes-db``` container to this network

---
### Dockerfile of multistage-build
---
Multi-staged build. The first stage is used for building and installing the dependencies using node-gyp and the second stage is for running the application.
```
# stage one
FROM node:lts-alpine as builder

# install dependencies for node-gyp
RUN apk add --no-cache python make g++

WORKDIR /app

COPY ./package.json .
RUN npm install --only=prod

# stage two
FROM node:lts-alpine

EXPOSE 3000
ENV NODE_ENV=production

USER node
RUN mkdir -p /home/node/app
WORKDIR /home/node/app

COPY . .
COPY --from=builder /app/node_modules  /home/node/app/node_modules

CMD [ "node", "bin/www" ]
```
To build the image from the Dockerfile:
```
docker image build --tag notes-api .
```
Before you run a container using this image, make sure the database container is running, and is attached to the notes-api-network.
```
docker container inspect notes-db
```
Once you're assured that everything is in place:
```
docker container run \
    --detach \
    --name=notes-api \
    --env DB_HOST=notes-db \
    --env DB_DATABASE=notesdb \
    --env DB_PASSWORD=secret \
    --publish=3000:3000 \
    --network=notes-api-network \
    notes-api
```
The container is running now. You can visit ``` http://127.0.0.1:3000/``` to see the API in action.

---
### Execute Commands in a Running Container
---
For executing a command inside a running container:
```
docker container exec <container identifier> <command>

docker container exec notes-api npm run db:migrate
```
In cases where you want to run an interactive command inside a running container, you'll have to use the -it flag. As an example, if you want to access the shell running inside the notes-api container, you can execute following the command:
```
docker container exec -it notes-api sh
```

---
### Write Management Scripts in Docker
---
Managing a multi-container project along with the network and volumes and stuff means writing a lot of commands. To simplify the process:   
- ``` boot.sh ``` - Used for starting the containers if they already exist.
```
#!/bin/bash
set -e

API_CONTAINER_NAME="notes-api"
DB_CONTAINER_NAME="notes-db"

if docker container ls --all | grep -q $DB_CONTAINER_NAME;
then
  printf "starting db container --->\n"
  docker container start $DB_CONTAINER_NAME;
  printf "db container started --->\n"
else
  printf "db container not found --->\n"
fi

printf "\n"

if docker container ls --all | grep -q $API_CONTAINER_NAME;
then
  printf "starting api container --->\n"
  docker container start $API_CONTAINER_NAME;
  printf "api container started --->\n"
else
  printf "api container not found --->\n"
fi

printf "\n"

printf "boot script finished\n\n"
```

- ```build.sh``` - Creates and runs the containers. It also creates the images, volumes, and networks if necessary.
```
#!/bin/bash
set -e

NETWORK_NAME="notes-api-network"
DB_CONTAINER_VOLUME_NAME="notes-db-data"
API_IMAGE_NAME="notes-api"
API_CONTAINER_NAME="notes-api"
DB_CONTAINER_NAME="notes-db"

DB_NAME="notesdb"
DB_PASSWORD="secret"

if docker network ls | grep -q $NETWORK_NAME;
then
  printf "network found --->\n"
else
  printf "creating network --->\n"
  docker network create $NETWORK_NAME;
  printf "network created --->\n"
fi

printf "\n"

if docker volume ls | grep -q $DB_CONTAINER_VOLUME_NAME;
then
  printf "volume found --->\n"
else
  printf "creating volume --->\n"
  docker volume create $DB_CONTAINER_VOLUME_NAME;
  printf "volume created --->\n"
fi

printf "\n"

printf "starting db container --->\n"
if docker container ls --all | grep -q $DB_CONTAINER_NAME;
then
  docker container start $DB_CONTAINER_NAME
else
  docker container run \
    --detach \
    --volume $DB_CONTAINER_VOLUME_NAME:/var/lib/postgresql/data \
    --name=$DB_CONTAINER_NAME \
    --env POSTGRES_DB=$DB_NAME \
    --env POSTGRES_PASSWORD=$DB_PASSWORD \
    --network=$NETWORK_NAME \
    postgres:12;
fi
printf "db container started --->\n"

printf "\n"

cd api;
printf "creating api image --->\n"
docker image build . --tag $API_IMAGE_NAME;
printf "api image created --->\n"
printf "starting api container --->\n"
if docker container ls --all | grep -q $API_CONTAINER_NAME;
then
  docker container start $API_CONTAINER_NAME
else
  docker container run \
      --detach \
      --name=$API_CONTAINER_NAME \
      --env DB_HOST=$DB_CONTAINER_NAME \
      --env DB_DATABASE=$DB_NAME \
      --env DB_PASSWORD=$DB_PASSWORD \
      --publish=3000:3000 \
      --network=$NETWORK_NAME \
      $API_IMAGE_NAME;
  docker container exec $API_CONTAINER_NAME npm run db:migrate;
fi
printf "api container started --->\n"

cd ..
printf "\n"

printf "build script finished\n\n"
```

- ```destroy.sh``` - Removes all containers, volumes and networks associated with this project.
```
#!/bin/bash
set -e

NETWORK_NAME="notes-api-network"
DB_CONTAINER_VOLUME_NAME="notes-db-data"
API_CONTAINER_NAME="notes-api"
DB_CONTAINER_NAME="notes-db"

if docker container ls --all | grep -q $API_CONTAINER_NAME;
then
  printf "removing api container --->\n"
  docker container rm $API_CONTAINER_NAME;
  printf "api container removed --->\n"
else
  printf "api container not found --->\n"
fi

printf "\n"

if docker container ls --all | grep -q $DB_CONTAINER_NAME;
then
  printf "removing db container --->\n"
  docker container rm $DB_CONTAINER_NAME;
  printf "db container removed --->\n"
else
  printf "db container not found --->\n"
fi

printf "\n"

if docker volume ls | grep -q $DB_CONTAINER_VOLUME_NAME;
then
  printf "removing db data volume --->\n"
  docker volume rm $DB_CONTAINER_VOLUME_NAME;
  printf "db data volume removed --->\n"
else
  printf "db data volume not found --->\n"
fi

printf "\n"

if docker network ls | grep -q $NETWORK_NAME;
then
  printf "removing network --->\n"
  docker network rm $NETWORK_NAME;
  printf "network removed --->\n"
else
  printf "network not found --->\n"
fi

printf "\n"

printf "destroy script finished\n\n"
```
- stop.sh - Stops all running containers.
```
#!/bin/bash
set -e

API_CONTAINER_NAME="notes-api"
DB_CONTAINER_NAME="notes-db"

if docker container ls | grep -q $API_CONTAINER_NAME;
then
  printf "stopping api container --->\n"
  docker container stop $API_CONTAINER_NAME;
  printf "api container stopped --->\n"
else
  printf "api container not found --->\n"
fi

printf "\n"

if docker container ls | grep -q $DB_CONTAINER_NAME;
then
  printf "stopping db container --->\n"
  docker container stop $DB_CONTAINER_NAME;
  printf "db container stopped --->\n"
else
  printf "db container not found --->\n"
fi

printf "\n"

printf "shutdown script finished\n\n"
```

- and ```Makefile``` - that contains four targets named start, stop, build and destroy, each invoking the previously mentioned shell scripts.
```
#################
## Production ##
################
start:
	./boot.sh
build:
	./build.sh
stop:
	./shutdown.sh
destroy: stop
	./destroy.sh

##################
## Development ##
#################
dev-start:
	docker-compose up --detach
dev-build:
	docker-compose up --detach --build; docker-compose exec api npm run db:migrate
dev-shell:
	docker-compose exec api bash
dev-stop:
	docker-compose stop
dev-destroy:
	docker-compose down --volume
```
If the container is in a running state in your system, executing ```make stop``` should stop all the containers. Executing ```make destroy``` should stop the containers and remove everything. Make sure you're running the scripts inside the app directory.

if permission error:
```
chmod +x boot.sh build.sh destroy.sh shutdown.sh
```
