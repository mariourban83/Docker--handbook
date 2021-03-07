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

---
### Multi-Staged Builds in Docker
---
In development mode the ```npm run``` serve command starts a development server that serves the application to the user. That server not only serves the files but also provides the hot reload feature.

In production mode, the ```npm run build``` command compiles all your JavaScript code into some static HTML, CSS, and JavaScript files. To run these files you don't need node or any other runtime dependencies. All you need is a server like nginx for example.

- Use node image as the base and build the application.
- Copy the files created using the node image to an nginx image.
- Create the final image based on nginx and discard all node related stuff.

This way your image only contains the files that are needed and becomes really handy.

This approach is a multi-staged build. 

Create new Dockerfile:
```
FROM node:lts-alpine as builder

WORKDIR /app

COPY ./package.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:stable-alpine

EXPOSE 80

COPY --from=builder /app/dist /usr/share/nginx/html
```
run the container with 
```
docker container run \
    --rm \
    --detach \
    --name hello-dock-prod \
    --publish 8080:80 \
    hello-dock:prod
```
The running application should be available on http://127.0.0.1:8080 

Multi-staged builds can be very useful if you're building large applications with a lot of dependencies. If configured properly, images built in multiple stages can be very optimized and compact.

---
### Ignoring Unnecessary files with ```.dockerignore```
---
The ```.dockerignore``` file contains a list of files and directories to be excluded from image builds.
```
.git
*Dockerfile*
*docker-compose*
node_modules
```
This ```.dockerignore``` file has to be in the build context. Files and directories mentioned here will be ignored by the COPY instruction. But if you do a bind mount, the .dockerignore file will have no effect. 