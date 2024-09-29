# Deploy laravel application with Docker Swarm

## Table of Contents

- [Introduction](#introduction)
- [Docker Services](#docker-services)
- [Requirements](#requirements)
- [Stop & Remove all the containers (optional)](#stop--remove-all-the-containers-optional)
- [File Overview](#file-overview)
- [Installation](installation#)

## Introduction

Create multiple VM (at least two or three) to form your Swarm. Choose an image with Docker pre-installed (e.g., Ubuntu 22.04).
Take note of their IP addresses, as you will use them to connect and configure your Swarm.

## Docker Services

- nginx
- app
- worker
- database
- redis
- certbot
- mailhog

## Requirements

- Docker
- Docker Compose
- Docker Swarm

If you have installed `Docker` and `Docker Compose`, follow these steps: [docker](https://docs.docker.com/engine/install/) [docker-compose](https://docs.docker.com/compose/install/)

## Stop & Remove all the containers (optional)

To stop and remove all Docker containers, you can run the following commands:

```shell
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)
```

## File Overview

The project structure looks like this:

```shell
Docker-Swarm
├── .docker
├── app
│   └── all app files
├── .dockerignore
├── .gitignore
├── .env
├── docker-compose.yml.example
├── README.md
└── make-ssl.sh.example
```

## installation docker leader(Manager) and worker (if not already installed)

```shell
sudo apt update
sudo apt install docker.io -y
```

Make sure Docker is running:

```shell
sudo systemctl enable docker
sudo systemctl start docker
```

## Initialize the Docker Swarm leader(Manager) and worker

On the manager node (one of your vm or instance), initialize the Docker Swarm:

```shell
docker swarm init --advertise-addr <MANAGER_IP>
```

Replace <MANAGER_IP> with the public IP of your vm or instance.

This command will give you a join token, something like this:

```shell
docker swarm join --token <TOKEN> <MANAGER_IP>:2377
```

## Join Worker Nodes to the Swarm

SSH into the other vm or instance (worker nodes) and run the join command provided from the manager node:

```shell
docker swarm join --token <TOKEN> <MANAGER_IP>:2377
```

Replace <TOKEN> and <MANAGER_IP> with the values from the output when you initialized the Swarm.

## Verify the Swarm Nodes MANAGER

```shell
docker node ls
```

This will show all the nodes in your Swarm, including the manager and workers.

```shell
root@ubuntu-1:# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
8n24p6bn5gmcql8zp *   ubuntu-1   Ready     Active         Leader           27.3.1
ljb1xqaermbz6ib0h0    ubuntu-2   Ready     Active                          27.3.1

```

## Create a Docker Swarm Overlay Network

Now that Swarm is initialized, you need to create an overlay network. This network will enable your containers to communicate with each other across different nodes in the Swarm.

Run the following command to create the network:

```shell
docker network create -d overlay --attachable my_overlay_network
```

- -d overlay: Specifies that it is an overlay network.
- --attachable: Allows standalone containers to attach to this network.

- This creates a new network called `my_overlay_network` that will span across all nodes in the Swarm.
  Ensure that the `docker-compose.yml` specifies external: true under networks to use the external Swarm network.

## Create Volumes in Docker Swarm

For Docker Swarm, volume handling is similar to standalone `Docker Compose`. The volumes you defined in the `docker-compose.yml (e.g., ./app, ./.data/redis, etc.)` will be mounted automatically when you deploy the stack.

If you need shared volumes across multiple nodes, ensure you're using a distributed storage solution like `NFS or a Docker volume plugin like GlusterFS. Otherwise, local volumes` will be limited to the node where the container is running.

## Setup NFS Docker volume all node manager and worker

Step 1: Set Up NFS Server on Leader Node

- Install NFS Server on the Leader Node:

```shell
sudo apt update
sudo apt install nfs-kernel-server -y
```

- Create the NFS Shared Directory:

Create the directory /`sites/zcart/app` to share with the other swarm nodes.

```shell
sudo mkdir -p /sites/zcart/app
```

- Give Proper Permissions:

Set appropriate ownership and permissions for the shared directory.

```shell
sudo chmod 777 /sites/zcart/app
```

- Configure NFS Exports:

```shell
sudo nano /etc/exports

/sites/zcart/app managerip(rw,sync,no_subtree_check,insecure) workerip(rw,sync,no_subtree_check,insecure)
```

- Export the Shared Directory:

Export the shared directory using the following command:

```shell
sudo exportfs -a
```

- Start the NFS Server:

Start the NFS service and ensure it starts on boot:

```shell
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

Step 2: Mount NFS on Worker Nodes

On each worker node (and the leader node itself, if needed), you need to mount the NFS share.

Install NFS Client on All Worker Nodes:

On each node, run the following commands:

```shell
sudo apt update
sudo apt install nfs-kernel-server -y
```

- Create the NFS Shared Directory:

Create the directory /`sites/zcart/app` to share with the other swarm nodes.

```shell
sudo mkdir -p /sites/zcart/app
```

- Give Proper Permissions:

Set appropriate ownership and permissions for the shared directory.

```shell
sudo chmod 777 /sites/zcart/app
```

Mount the NFS Share:

- Manually mount the NFS share from the leader node to the worker node:

```shell
sudo mount managerip:/sites/zcart/app /sites/zcart/app
```

- Verify the Mount:

```shell
df -h
```

Step 3: Update Your `docker-compose.yml` File

Now that the NFS share is set up, you need to configure your docker-compose.yml file to use it as a volume.

```shell
volumes:
  app_data:
    driver_opts:
      type: "nfs"
      o: "addr=157.230.221.85,rw,insecure,vers=3"
      device: ":/sites/zcart/app"
  db_data:
    external: true
```

## Deploy a Stack or Services to the Swarm laravel application

Once your network, volume and Swarm are set up, you can deploy services to the Swarm. Here’s an example using Docker Compose to deploy services on the Swarm.

- Create a docker-compose.yml file (on the manager node):

```shell
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    volumes:
      - ./.docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./.docker/nginx/app/FPM/dev.conf:/etc/nginx/conf.d/default.conf
      - ./.data/nginx/logs:/var/log/nginx
      - app_data:/var/www/app
      - ./.data/certs/certbot/conf:/etc/letsencrypt # uncomment when production deploy
      - ./.data/certs/certbot/www:/var/www/certbot # uncomment when production deploy

    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app
    networks:
      - my_overlay_network
    environment:
      - X_SERVER_TYPE=nginx
    deploy:
      mode: replicated
      replicas: 1 # Single replica for the Nginx service (as a reverse proxy)
      restart_policy:
        condition: on-failure

  app:
    build:
      context: .
      dockerfile: ./.docker/app/FPM/${PHP_VERSION}/Dockerfile
    volumes:
      - app_data:/var/www/app
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 3 # 3 replicas for load balancing
      restart_policy:
        condition: on-failure
    environment:
      - X_SERVER_TYPE=app

  worker:
    build:
      context: .
      dockerfile: ./.docker/worker/FPM/${PHP_VERSION}/Dockerfile
    volumes:
      - app_data:/var/www/app
    command: ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 3 # 3 replicas for the worker service
      restart_policy:
        condition: on-failure
    environment:
      - X_SERVER_TYPE=worker

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - ./.data/redis:/data
    entrypoint: redis-server --appendonly yes
    restart: always
    networks:
      - my_overlay_network

  database:
    container_name: database
    image: postgres:16
    environment:
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE}
      PGDATA: /data/postgres
    volumes:
      - db_data:/data/postgres
    ports:
      - "5432:5432"
    restart: always
    networks:
      - my_overlay_network
  certbot:
    image: certbot/certbot
    container_name: certbot
    restart: unless-stopped
    volumes:
      - ./.data/certs/certbot/conf:/etc/letsencrypt
      - ./.data/certs/certbot/www:/var/www/certbot
    networks:
      - my_overlay_network
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  app_data:
    driver_opts:
      type: "nfs"
      o: "addr=(Manager ip for nfs),rw,insecure,vers=3"
      device: ":/sites/zcart/app"
  db_data:
    external: true

networks:
  my_overlay_network:
    external: true
```

## Key Points

- Mounting the Volume: The volume `app_data` is defined as an external volume in the volumes section. It is then mounted into the `nginx, app, and worker services` at `/var/www/app`. You can modify this path according to where you want the volume to be mounted inside the container.

- Overlay Network: The network `my_overlay_network` is defined as `external`, meaning it's an existing network you've already created with Docker Swarm `(docker network create -d overlay --attachable my_overlay_network)`. This ensures that the services communicate over the Swarm overlay network.

## How the build Works in the Compose File

- Context: This is the directory where your Dockerfile and relevant build files are located. In this case, `context: .` points to the current directory where your Docker Compose file is located.

Dockerfile: Specifies the relative path to the Dockerfile within the `context.` In your example, it's using ${PHP_VERSION} as a dynamic value in the path, so it will look for the Dockerfile in .docker/app/FPM/${PHP_VERSION}/Dockerfile and .docker/worker/FPM/${PHP_VERSION}/Dockerfile.

Building the Image: When you run docker-compose up or docker stack deploy, Docker will:

- Look at the build section for each service.
- Use the context directory as the build environment.
- Use the specified dockerfile to build the Docker image.
- Create a local image (with a name automatically assigned based on the Compose service name) to be used by the container.

## How to Trigger the Build

You can trigger the build process in two ways:

1. Using `docker-compose up --build`

```shell
docker-compose up --build
```

This command will build all services with the build option specified in your `docker-compose.yml` and start them.

2. Using `docker-compose build`
   You can also explicitly run the build step without starting the containers by using:

```shell
docker-compose build
```

You can specify a particular service to build:

```shell
docker-compose build app
```

**Note**: Build the Images Manually
Before deploying your stack, you need to manually build your images and push them to a Docker registry (e.g., Docker Hub or a private registry).

- For example:

```shell
docker tag your-app-image:latest your-dockerhub-username/your-app-image:latest
docker push your-dockerhub-username/your-app-image:latest
```

## Modify Your docker-compose.yml

Update your `docker-compose.yml` file to remove the build section and replace it with the image section, referring to the images you just pushed to your registry.

I have a create a new docker-compose-swarm.yml.
Updated docker-compose-swarm.yml:

```shell
volumes:
  app_data:
    driver_opts:
      type: "nfs"
      o: "addr=(Manager ip for nfs),rw,insecure,vers=3"
      device: ":/sites/zcart/app"
  db_data:
    external: true

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    volumes:
      - ./.docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./.docker/nginx/app/FPM/dev.conf:/etc/nginx/conf.d/default.conf
      - ./.data/nginx/logs:/var/log/nginx
      - app_data:/var/www/app
      - ./.data/certs/certbot/conf:/etc/letsencrypt # uncomment when production deploy
      - ./.data/certs/certbot/www:/var/www/certbot # uncomment when production deploy
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - app
    networks:
      - my_overlay_network
    environment:
      - X_SERVER_TYPE=nginx
    deploy:
      mode: replicated
      replicas: 1 # Single replica for the Nginx service (as a reverse proxy)
      restart_policy:
        condition: on-failure

  app:
    image: devops/zcart-app:latest  # Use the pushed image
    volumes:
      - app_data:/var/www/app
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 3 # 3 replicas for load balancing
      restart_policy:
        condition: on-failure
    environment:
      - X_SERVER_TYPE=app

  worker:
    image: devops/zcart-worker:latest  # Use the pushed image
    volumes:
      - app_data:/var/www/app
    command: ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 3 # 3 replicas for the worker service
      restart_policy:
        condition: on-failure
    environment:
      - X_SERVER_TYPE=worker

  redis:
    image: redis:latest
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - ./.data/redis:/data
    entrypoint: redis-server --appendonly yes
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 1 # Set replica to 1 for Redis
      restart_policy:
        condition: on-failure

  database:
    container_name: database
    image: postgres:16
    environment:
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE}
      PGDATA: /data/postgres
    volumes:
      - db_data:/data/postgres
    ports:
      - "5432:5432"
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 1 # Set replica to 1 for the database
      restart_policy:
        condition: on-failure

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./.data/certs/certbot/conf:/etc/letsencrypt
      - ./.data/certs/certbot/www:/var/www/certbot
    networks:
      - my_overlay_network
    deploy:
      mode: replicated
      replicas: 1 # Set replica to 1 for Certbot
      restart_policy:
        condition: on-failure
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

networks:
  my_overlay_network:
    external: true
```

Step 3: Deploy the Stack

- After modifying the `docker-compose-swarm.yml` file, redeploy your stack:

```shell
docker stack deploy -c docker-compose-swarm.yml laravel_zcart
```

Step 5: Verify the Setup

Check if the volume is properly mounted and shared across all nodes:

```shell
docker node ls
docker node inspect <node_id> --pretty
docker service ls
docker stack services laravel_zcart
docker service ps laravel_zcart_app
docker service logs <service_name>
docker stats
docker service scale <service_name>=<replica_count>
```

check running service and replica for manager

```shell
root@ubuntu-1:/sites/zcart# docker service ls
ID             NAME                     MODE         REPLICAS   IMAGE                        PORTS
w9yzc542t6r6   laravel_zcart_app        replicated   5/5        sayad1/zcart-app:latest
lnnql8lu98r3   laravel_zcart_certbot    replicated   1/1        certbot/certbot:latest
luwoh0jpapfo   laravel_zcart_database   replicated   1/1        postgres:16                  *:5432->5432/tcp
g3ncecpcocf2   laravel_zcart_nginx      replicated   1/1        nginx:latest                 *:80->80/tcp, *:443->443/tcp
wiqjny28bosa   laravel_zcart_redis      replicated   1/1        redis:latest                 *:6379->6379/tcp
ll59selz4mwy   laravel_zcart_worker     replicated   3/3        sayad1/zcart-worker:latest
root@ubuntu-1:/sites/zcart#
```

check running service and replica for manager

```shell
root@ubuntu-1:/sites/zcart# docker ps -a
CONTAINER ID   IMAGE                        COMMAND                  CREATED       STATUS       PORTS             NAMES
4b60cd56fae6   sayad1/zcart-app:latest      "/usr/bin/startx.sh"     2 hours ago   Up 2 hours   9000/tcp          laravel_zcart_app.4.o1zwdztujv3377krhj10m2y3i
ac318ffced9d   postgres:16                  "docker-entrypoint.s…"   2 hours ago   Up 2 hours   5432/tcp          laravel_zcart_database.1.i72hvtvsjv8mq7qh1tpzlelo0
dc008c80959f   nginx:latest                 "/docker-entrypoint.…"   2 hours ago   Up 2 hours   80/tcp            laravel_zcart_nginx.1.j72s06hkoxyjkb92wax3qpgyr
fb5ad5f55778   sayad1/zcart-worker:latest   "docker-php-entrypoi…"   2 hours ago   Up 2 hours   9000/tcp          laravel_zcart_worker.1.xnfs1h40lr0axdksh8v4oqjxq
2b146622ee34   sayad1/zcart-app:latest      "/usr/bin/startx.sh"     2 hours ago   Up 2 hours   9000/tcp          laravel_zcart_app.1.td1h5rc8x1d1fudwezxsr3u3m
95f06aacf448   certbot/certbot:latest       "/bin/sh -c 'trap ex…"   2 hours ago   Up 2 hours   80/tcp, 443/tcp   laravel_zcart_certbot.1.ckzalo89skl2e8x6it06m9gvj
ba4d86d3054d   redis:latest                 "redis-server --appe…"   2 hours ago   Up 2 hours   6379/tcp          laravel_zcart_redis.1.xzfhq4citulleapr269d2ybrm
root@ubuntu-1:/sites/zcart#
```

## worker

```shell
4ddd4402748c   sayad1/zcart-worker:latest   "docker-php-entrypoi…"   2 hours ago   Up 2 hours               9000/tcp   laravel_zcart_worker.2.2528xfr0pzmbsqqz740djpypu
528658c02fad   sayad1/zcart-worker:latest   "docker-php-entrypoi…"   2 hours ago   Up 2 hours               9000/tcp   laravel_zcart_worker.3.ubwg03op5fb3o69u17gqspi8b
f1b2fc55cba6   sayad1/zcart-app:latest      "/usr/bin/startx.sh"     2 hours ago   Up 2 hours               9000/tcp   laravel_zcart_app.3.x0u7fjiuui39qisu2863fmwzx
7b4382bd1b72   sayad1/zcart-app:latest      "/usr/bin/startx.sh"     2 hours ago   Up 2 hours               9000/tcp   laravel_zcart_app.2.719kffqziyfbl9civul0jk1sf
root@ubuntu-2:/sites/zcart#
```
