# Docker Setup For Laravel

This is a complete docker setup for developing, running and deploying a laravel application without going through the tiring process of installing php, nginx and a ton of packages required to run laravel.

You can just have the environment ready for development with two commands

1. `docker-compose run --rm composer create-project laravel/laravel .`
2. `docker-compose up -d server`

You will have your laravel project in src folder and you can see it on `localhost:8080`

In case you are cusrious and want to understand the configurations

Here is a detailed explanation of all the containers that we launch here.

## Configurations

### PHP

This is about main php container.

We have some extra configs so we have a separate docker file for php

```docker
FROM php:8.1-fpm-alpine

# This is the standard working directory for servers
WORKDIR /var/www/html

# To install pdo and pdo_mysql to talk to mysql
RUN docker-php-ext-install pdo pdo_mysql

# Adding a user
RUN addgroup -g 1000 laravel && adduser -G laravel -g laravel -s /bin/sh -D laravel

# Using that created user
USER laravel
```

Then coming to the php part in docker-compose file, we just set the build to the above docker file and add a bind mount for code reflection to src.

**Note:** We are adding :delegated to bind mount so that the changes are processed in batches to improve performance.

**Note:** PHP official image exposes 9000 port. Thus we have 9000 port in nginx

### Nginx

```C
  // set the image to nginx
  server:
    image: 'nginx:stable-alpine'
    //Set the source code bind mount and nginx conf bind mount
    volumes:
      - ./src:/var/www/html
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    // Port Mapping
    ports:
      - "8080:80"
    // Some nginx environmnet variables
    environment:
      - NGINX_HOST=foobar.com
      - NGINX_PORT=80
    // Dependecies
    depends_on:
      - php
      - mysql
```

- nginx.conf has got `fastcgi_pass php:9000;`, here we have php:9000 because all containers are in same network and our php container is identified by name by docker thus we have php:9000 instead of IP:9000.

- We are also adding /src as a bind mount to /var/www/html as server also needs to know about our source code. At nginx.conf `root /var/www/html/public;`

### MYSQL

```C
  mysql:
    // Using the official mysql image
    image: mysql:5.7
    //Pointing it to env file with username, db and password which you need to edit
    env_file:
      - ./env/mysql.env
```
**Note:** Dont't forget to set a default db, user, password and root password in the env/mysql.env file

### Composer

Composer is used to generate laravel code and is thus here as a utility container.


The dockerfile goes like this

```docker
# Official composer image
FROM composer:latest

# Adding users
RUN addgroup -g 1000 laravel && adduser -G laravel -g laravel -s /bin/sh -D laravel

# Using that user so that we don't get any persmission issues
USER laravel

# Setting a workdir
WORKDIR /var/www/html

# Giving an entrypoint so that we can append to it while running a container
ENTRYPOINT ["composer", "--ignore-platform-reqs"]

```

And then the compose part goes like this.
```C
  composer:
    // Composer custom dockerfile which will be used to build image
    build: 
      context: ./dockerFiles
      dockerfile: composer.dockerfile
    // Adding source code bind mount
    volumes:
      - ./src:/var/www/html

```
Adding a bind mount from src to /var/www/html so that we can have a mirror on host.


### Artisan

Artisan is another utility used in laravel development. Added it as well to make it available.

The compose part goes like this

```C
  artisan:
    // Artisan can be built from the same php dockerfile
    build: 
      context: ./dockerFiles
      dockerfile: php.dockerfile
    volumes:
    // Adding source code bind mount to make source code available to it
      - ./src:/var/www/html
    // Adding a custom entry point so that we can execute artisan by appending specific commands while running.
    entrypoint: ["php", "/var/www/html/artisan"]
```

### npm

We also have simple npm container in here which is required occassionally.

It is a straightforward npm setup.

Docker compose for npm goes this way

```C
  npm:
    // npm is build on top of node
    image: node
    // We set a working dir here itself
    working_dir: /var/www/html/
    // Entry point is npm as so that we can append any npm comamnd to docker run
    entrypoint: ["npm"]
    // Making source code available by bind mount
    volumes:
      - ./src:/var/www/html
```
I did not write a custom dockerfile for npm as it doesn't need anything more complex than just a working dir and entrypoint which are available in compose.

## How To Use?

1. Git clone
2. Navigate into root of the project
3. Run `docker-compose run --rm composer create-project laravel/laravel .`. This will setup a laravel project and will also reflect the same into src folder of your project.
4. Edit the sr/.env file and add db configs
5. Run `docker-compose up -d server`. Only server, as server depends on php and mysql and this commands thus starts php and mysql. **Note:** Add `--build` to check for changes in dockerfiles.

### To Use Utility Containers Like Artisan

1. Run `docker-compose run --rm artisan migrate`. You just add migrate at end as we have already given an entry point in docker compose file.


## For Production

1. In production as we don't have bind mounts, we have to have the source code in the image itself.
2. For that, after development, you can replace dockerFiles folder with the prod/dockerFiles folder.
3. Replace docker-compose.yaml file with prod/docker-compose.yaml
4. These prod files remove bind mounts and copy the source code into the respective images.

Hope it help.

Do star the repository if it does :-)