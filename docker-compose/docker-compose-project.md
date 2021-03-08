# Compose Projects Using Docker-Compose

An easier way to manage multi-container projects is called Docker Compose.   
According to the Docker documentation:

" Compose is a tool for defining and running multi-container Docker applications. With Compose, you use a YAML file to configure your application’s services. Then, with a single command, you create and start all the services from your configuration."

Although Compose works in all environments, it's more focused on development and testing. Using Compose on a production environment is not recommended at all !

---
### Docker Compose Basics
---
Create Dockerfile :
```
# stage one
FROM node:lts-alpine as builder

# install dependencies for node-gyp
RUN apk add --no-cache python make g++

WORKDIR /app

COPY ./package.json .
RUN npm install

# stage two
FROM node:lts-alpine

ENV NODE_ENV=development

USER node
RUN mkdir -p /home/node/app
WORKDIR /home/node/app

COPY . .
COPY --from=builder /app/node_modules /home/node/app/node_modules

CMD [ "./node_modules/.bin/nodemon", "--config", "nodemon.json", "bin/www" ]
```
Just like the Docker daemon uses a Dockerfile for building images, Docker Compose uses a docker-compose.yaml file to read service definitions from.   

Create docker-compose.yaml file :
```
version: "3.8"

services: 
    db:
        image: postgres:12
        container_name: notes-db-dev
        volumes: 
            - notes-db-dev-data:/var/lib/postgresql/data
        environment:
            POSTGRES_DB: notesdb
            POSTGRES_PASSWORD: secret
    api:
        build:
            context: ./api
            dockerfile: Dockerfile.dev
        image: notes-api:dev
        container_name: notes-api-dev
        environment: 
            DB_HOST: db ## same as the database service name
            DB_DATABASE: notesdb
            DB_PASSWORD: secret
        volumes: 
            - /home/node/app/node_modules
            - ./api:/home/node/app
        ports: 
            - 3000:3000

volumes:
    notes-db-dev-data:
        name: notes-db-dev-data
```
Every valid docker-compose.yaml file starts by defining the file version.   
Blocks in an yaml file above are defined by indentation. In the yaml file:   

- The ```services``` block holds the definitions for each of the services or containers in the application. db and api are the two services that comprise this project.   

- The ```db``` block defines a new service in the application and holds necessary information to start the container. Every service requires either a pre-built image or a Dockerfile to run a container. For the db service we're using the official PostgreSQL image.   
  
- Unlike the db service, a pre-built image for the ```api``` service doesn't exist. So we'll use the Dockerfile.dev file.   

- The ```volumes``` block defines any name volume needed by any of the services. At the time it only enlists notes-db-dev-data volume used by the db service.   

The definition code for the db service:

- The ```image``` key holds the image repository and tag used for this container. We're using the postgres:12 image for running the database container.   

- The ```container_name``` indicates the name of the container. By default containers are named following < project directory name >_< service name> syntax. You can override that using container_name.   

- The ```volumes``` array holds the volume mappings for the service and supports named volumes, anonymous volumes, and bind mounts. The syntax < source >:< destination > is identical to what you've seen before.   

- The ```environment``` map holds the values of the various environment variables needed for the service.

Definition code for the api service:

- The ```api``` service doesn't come with a pre-built image. Instead it has a build configuration. Under the build block we define the context and the name of the Dockerfile for building an image. You should have an understanding of context and Dockerfile by now so I won't spend time explaining those.   

- The ```image``` key holds the name of the image to be built. If not assigned, the image will be named following the < project directory name >_< service name > syntax.   

- Inside the ```environment``` map, the DB_HOST variable demonstrates a feature of Compose. That is, you can refer to another service in the same application by using its name. So the db here, will be replaced by the IP address of the api service container. The DB_DATABASE and DB_PASSWORD variables have to match up with POSTGRES_DB and POSTGRES_PASSWORD respectively from the db service definition.   

- In the ```volumes``` map, you can see an anonymous volume and a bind mount described. The syntax is identical to what you've seen in previous sections.   

- The ports map defines any port mapping. The syntax, < host port >:< container port > is identical to the --publish option you used before.

The code for the volumes :   

Any named volume used in any of the services has to be defined here. If you don't define a name, the volume will be named following the < project directory name >_< volume key> and the key here is db-data.

---
### Start Services in Docker Compose 
---

There are a few ways of starting services defined in a YAML file. The first command is the ```up``` command. The up command builds any missing images, creates containers, and starts them in one go.
```
docker-compose --file docker-compose.yaml up --detach
```
Apart from the the ```up``` command there is the ```start``` command. The main difference between these two is that the start command doesn't create missing containers, only starts existing containers. It's basically the same as the container start command.

The ```--build``` option for the up command forces a rebuild of the images. There are some other options for the up command, see the docker official docs.

---
### Execute Commands Inside a Running Service in Docker Compose
---
Just like the ```container exec``` command, there is an ```exec``` command for docker-compose.
```
docker-compose exec <service name> <command>

docker-compose exec api npm run db:migrate
```
Unlike the container exec command, you don't need to pass the ```-it``` flag for interactive sessions, docker-compose does that automatically.

---
### Access Logs from a Running Service in Docker Compose
---
```
docker-compose logs <service name>

// To access the logs from the api service:
docker-compose logs api
```

---
### Stop Services in Docker Compose
---
To stop services, there are two approaches that you can take. The first one is the ```down``` command. The down command stops all running containers and removes them from the system. It also removes any networks:
```
docker-compose down --volumes
```
The ```--volumes``` option indicates that you want to remove any named volume(s) defined in the volumes block.

Another command for stopping services is the ```stop``` command which functions identically to the container stop command. It stops all the containers for the application and keeps them. These containers can later be started with the ```start``` or ```up``` command.