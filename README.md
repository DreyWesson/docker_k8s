
# Docker & Kubernetes

This project goes into details of how to containerize our application using Docker & Kubernetes.


## Tech Stack

**Others:** Docker, Kubernetes

**Client:** React

**Server:** Node, Express

  
## Docker Commands
This command creates and run a container from an image
```bash
    docker run <image_name>
```
This command lists all running container
```bash
    docker ps
```
This command shows all containers you've ever created
```bash
    docker ps --all
```
This clears out containers and freeup memory
```bash
    docker system prune
```
This shows the output of the container it was called with
```bash
    docker logs <container_id>
```
This stops a container with a grace period, else it kills it.
```bash
    docker stop <container_id>
```
This stops a container immediately with no grace period
```bash
    docker kill <container_id>
```
This helps u run cmd inside a container
```bash
    docker exec -it <container_id> <command>
```
This helps u run cmd inside a container and also keeps the shell open
```bash
    docker exec -it <container_id> sh
```
This works similarly as `exec` but you can't run any other process
```bash
    docker run -it <container_id> sh
```


## Dockerfile
====Build Phase====

`FROM node:alpine as builder` (use an existing docker image as base `as builder` serves as a tag)

`WORKDIR /app` (prevents pollution & name conflict in our container)

`COPY ./package.json ./` (if nothing changed prevent re-installation)

`RUN npm install` (download and install dependencies)

`COPY ./ ./` (copy current directory content into our container)

`RUN npm run build`

=====Run Phase=====

`FROM nginx`

`EXPOSE 80` (used by elasticbeanstalk to map traffic)

`COPY --from=builder /app/build /usr/share/nginx/html` (use `builder` tagged previously, and copy `/app/build` into `/usr/share/nginx/html`)



### Running Dockerfile
To run the Dockerfile, execute this CMD in the same directory as the Dockerfile
```bash
    docker build .
```
#### Tagging an image
To tag your image for easy reuse. Note that you'll use `docker_username/project_name` as your own `image_name` 
```bash
    docker build -t <image_name>:latest .
```
#### Run tagged image
To run your tagged image 
```bash
    docker run 
```
#### Port Mapping
The mapping of port helps u to route traffic to your container. 
To change your container_port, do change the port your `app.listening(xxxx,cb` too.
```bash
    docker run -p <source_port>:<container_port> <image_name>
```


## Dockerfile.dev
`FROM node:alpine` (use an existing docker image as base)

`WORKDIR /app` (prevents pollution & name conflict in our container)

`COPY ./package.json ./` (if nothing changed prevent re-installation)

`RUN npm install` (download and install dependencies)

`COPY ./ ./` (copy current directory content into our container)

`EXPOSE <port_no>` (maps traffic to port in the container)

`CMD ["npm", "start"]` (start the container CMD)

### Running Dockerfile.dev
Execute this CMD in the same directory as the Dockerfile.dev. 
Also, delete `node_modules` folder to prevent duplication
```bash
    docker build -f Dockerfile.dev .
```
### Docker volume
Maps a folder inside a container to a container outside a container. This is helpful in  
getting the latest changes(hot reloading) to the folder on our local machine
```bash
    docker run -p <port_no>:<port_no> -v /app/node_modules -v $(pwd):/app <image_id>
```
`-v /app/node_modules`: this instructs docker to just use node_modules as a placeholder  
and not to map node_modules in the container to the local(which doesn't even exist).

`-v $(pwd):/app`: the `:` instructs docker to map content in the `pwd` to `app` in the container

## Docker Compose
This file helps us to reduce repitition of CMDs and also communication between containers.

### Commands

`docker-compose up` (to run our image)

`docker-compose up -d` (to run our image in the background)

`docker-compose up --build` (to build and run image)

`docker-compose down` (to stop our container)
`docker_compose ps` (list running container)

### docker-compose.yml
```
version: "3" (specifies the version of docker-compose to use)
services: (lists type of services)
    redis-server: (service01)
        image: "redis" (the image to use in building the service)
    node-app: (service02)
        restart: always (when service stops/crashes)
        build: . (builds app in the current dir. works for Dockerfile not Dockerfile.dev)
        ports: 
            - "4001:8081"
    web:
        build: 
            context: . (specifies where we want files and folders to be pulled. if we had nested our react folder it would have been 'react-app' not '.')
            dockerfile: Dockerfile.dev (forces docker_compose to use the Dockerfile.dev)
        ports:
            - "3000:3000"
        volumes: (refer to Docker volume)
            - /app/node_modules
            - .:/app
    tests:
        build: 
            context: . (specifies where we want files and folders to be pulled. if we had nested our react folder it would have been 'react-app' not '.')
            dockerfile: Dockerfile.dev (forces docker_compose to use the Dockerfile.dev)
        volumes: (refer to Docker volume)
            - /app/node_modules
            - .:/app
        command: ["npm", "run", "test"] (overrides the default "npm start" CMD)
    server:
        build:
            docker: Dockerfile.dev
            context: ./server
        environment:
```

### Run different docker compose
[stackoverflow](https://stackoverflow.com/questions/47039538/docker-compose-for-production-and-development)

[stackoverflow](https://stackoverflow.com/questions/62122006/node-docker-compose-development-and-production-setup)

`docker-compose  -f docker-compose.yml -f docker-compose.prod.yml up -d`

`docker-compose  -f docker-compose.yml -f docker-compose.dev.yml up -d`

## Nginx

To start nginx

- Run `docker build .`
- Copy the image SHA
- Run `docker run -p 8080:80 <SHA>`

### nginx/default.conf
```
    upstream client { (we have an upstream that we'll call client)
        server client:3000; (its url is 3000)
    }

    upstream backend { (we have an upstream that we'll call backend)
        server backend:5000; (its url is 5000)
    }

    server { 
        (this helps to route all traffic from port 80 to 3000(for "/" traffic) and 5000(for "/api" traffic))
        
        listen 80;

        location / {
            proxy_pass http://client;
        }

        location /sockjs-node {
            (takes care of error that hampers app performance)
            proxy_pass http://client;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "Upgrade";
        }

        location /api {
            rewrite /api/(.*) /$1 break;  (this helps chop off the '/api' in the url)
            proxy_pass http://api;
        }

    }
```

### nginx/Dockerfile.dev
```
FROM nginx
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

## travis.yml
AWS set-up
```
sudo: required
services:
    - docker
before_install:
    - docker build -t <dreywesson/project_name> -f Dockerfile.dev .
scripts:
    - docker run <dreywesson/project_name> npm run test -- --coverage
deploy:
    provider: elasticbeanstalk (or any cloud service provider)
    region: "us-west-2" (region used in setting up our provider)
    app: "docker-react" (project_name)
    env: "<Docker-env>"
    bucket_name: "<string>" (generated by elasticbeanstalk)
    bucket_path: "docker-react" (project_name)
    on:
        branch: master
    access_key_id: $AWS_ACCESS_KEY
    secret_access_key:
    secure: "$AWS_SECRET_KEY
```

## Kubernetes
To run k8s yml files
```
kubectl apply -f <filename>
```  
To check configured pods 
```
kubectl get pods
```
To check services 
```
kubectl get services
```
You can't access k8s app in browser using localhost. You'll use IP address generated by
```
    minikube ip
```
#### Error handling docker-compose

This instructs our service on what to do if our container crashes/stops.

| Parameter        | Description (if container stops/crashes)                             |
| :--------------- | :--------------------------------------- |
| `"no"`             | don't restart                            |
| `always`         | always restart                           |
| `on-failure`     | only restart if stops with an error code |
| `unless-stopped` | always restart unless forcibly stopped   |



  
