# Containerize a JavaScript App

This is a very simple JavaScript project powered by the 'https://github.com/vitejs/vite' project.

A plan :
- Get a good base image for running JavaScript applications, like node.
- Set the default working directory inside the image.
- Copy the package.json file into the image.
- Install necessary dependencies.
- Copy the rest of the project files.
- Start the vite development server by executing npm run dev command.

Start with Dockerfile.dev
```
FROM node:lts-alpine

EXPOSE 3000

USER node

RUN mkdir -p /home/node/app

WORKDIR /home/node/app

COPY ./package.json .
RUN npm install

COPY . .

CMD [ "npm", "run", "dev" ]
```
Build the image, run it and visit 'http://127.0.0.1:3000'
```
docker image build --file Dockerfile.dev --tag hello-dock:dev .

docker container run \
    --rm \
    --detach \
    --publish 3000:3000 \
    --name hello-dock-dev \
    hello-dock:dev
```
---
### Work With Bind Mounts and Anonymous Volumes in Docker
---
Using bind mounts, you can easily mount one of your local file system directories inside a container. Instead of making a copy of the local file system, the bind mount can reference the local file system directly from inside the container.  

This way, any changes you make to your local source code will reflect immediately inside the container,  triggering the hot reload feature of the development server, if available. Changes made to the file system inside the container will be reflected on your local file system as well.  

An anonymous volume is identical to a bind mount except that you don't need to specify the source directory here.

```
docker container run \
    --rm \
    --detach \
    --publish 3000:3000 \
    --name hello-dock-dev \
    --volume $(pwd):/home/node/app \
    --volume /home/node/app/node_modules \
    containerize-js-app:dev  
```   
Here, Docker will take the entire node_modules directory from inside the container and tuck it away in some other directory managed by the Docker daemon on your host file system and will mount that directory as node_modules inside the container.