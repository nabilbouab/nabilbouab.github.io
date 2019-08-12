---
layout: post
title:  "Dockerize your application"
author: "Nabil"
---

You developed an application and you are not sure what is the best way to deploy it? Or do you have a deployment process but it is tedious and error prone and you are looking for a better and more reproducible way of deploying your software? If so, you are reading the right article.

## Why Docker?

There are plenty of reasons why you would want to run an application in Docker. The first one would be to have a reproducible environment in development and in production. And this is the common reason why people would use Docker.

However, I have another one: I want to avoid installing anything on our computer. I don't want to have rvm and swith between ruby versions on my computer. I don't want to have to run 20 commands and read 5 blog articles to install rails on my machine and have conflicts between different projects running on different versions. 

In a nutshell, I want to have my application running in a reproducible environment with one command line and this is what Docker gives us.  We want to make it easy for us to run locally and in production.

In this blog post, we took the example of a Rails application which communicates with a PostgreSQL database (but it could be any type of application communication with any database).

## Dockerize a Rails app

First, let's **create a Docker image for your Rails application** and a Docker image for the database and run it locally. Here is the Dockerfile used to run the rails application.

```
FROM ruby:2.0.0
RUN printf "deb http://archive.debian.org/debian/ jessie main\ndeb-src http://archive.debian.org/debian/ jessie main\ndeb http://security.debian.org jessie/updates main\ndeb-src http://security.debian.org jessie/updates main" > /etc/apt/sources.list
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
RUN mkdir /myapp
WORKDIR /myapp
COPY Gemfile /myapp/Gemfile
COPY Gemfile.lock /myapp/Gemfile.lock
RUN bundle install
COPY . /myapp

# Add a script to be executed every time the container starts.
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]
EXPOSE 3000

# Start the main process.
CMD ["rails", "server", "-b", "0.0.0.0"]
```
You need to copy and paste the content in a file named `Dockerfile` located at the same level as the `app` folder. Then, run `docker build . -t myapp`. At this stage, we created an image of our Ruby on Rails application.

For the PostgreSQL image, we will take the pre built one from Docker Hub which is the public artifact repository where anyone can push there own images.

## Create the docker-compose config file

**Now, we need to make the application communicate with the database**. We will use docker-compose in order to run multiple services: one of them will be the Ruby on Rails application, another one will be the database and the last one will be NGINX container. In order to describe our system, we create a compose file which is just a very straight forward yml file like so:

```yaml
version: '3'
services:
  db:
    image: mysql:5.7
    volumes:
      - ./tmp/db:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ######
      MYSQL_USER: ######
      MYSQL_PASSWORD: ######
      MYSQL_DATABASE: ######
  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
    volumes:
      - .:/myapp
    depends_on:
      - db
  nginx:
    build: ./nginx
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/ssl/dhparams:/etc/ssl/dhparams
      - /etc/letsencrypt:/etc/letsencrypt
    depends_on:
      - web
```

With this compose file, we spin up three services in three separate containers. One is nginx, the other one is our Ruby on Rails application and the last one is the PostgreSQL database.

Let's configure the Ruby on Rails application to be able to reach the database container to store data. When you run `docker-compose up`, a network and a local DNS are created: each service is reachable by its name within this network. We will change our configuration file in the rails application to cater for that:

```yaml
development:
  adapter: mysql2
  encoding: utf8
  host: db
  database: lamrani2
  pool: 5
  username: lamrani2
password: lamrani2
```

The host here is `db` which matches the name of our database service in our compose file. At this stage, the rails application container can communicate with the database container through the cdocker compose network.

At this stage, the application should be running locally and we should be able to store and retrieve data from it. We can then deploy it to a VPS on DigitalOcean for example and make sure it works there when running `docker-compose up`.