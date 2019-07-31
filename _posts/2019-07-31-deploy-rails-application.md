---
layout: post
title:  "Deploy rails application"
author: "Nabil"
---

If you want to know how to run your Ruby on Rails application with Docker compose and setup NGINX to have HTTPS, you are at the right place.

# Ruby on Rails on Docker

We want to avoid installing anything on our computer. I don't want to have rvm and swith between ruby versions. I don't want to have to run 20 commands and read 5 blog articles to install rails on my machine and have conflicts between different projects running on different version. I want to have my application running in a reproducible environment with one command line and this is what Docker gives us. 

First, let's **create a Docker image for the Rails application** and a Docker image for the database and run it locally. Here is the Dockerfile used to run the rails application.

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
You need to copy and paste the content in a file named `Dockerfile` and located at the same level as the `app` folder. Then, run `docker build . -t myapp`. 

**Now, we need to make the application communicate with the database**. We will use docker-compose in order to run multiple services. We need to create a compose file that will describe our system.

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

With this compose file, we spin up three services in three separate containers. Let's configure the rails application to be able to reach the database container to store data. When you run `docker-compose up`, a network and a local DNS are created: each service is reachable by its name within this network. We will change our configuration file in the rails application to cater for that:

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

# HTTPS support with NGINX

We need to secure all that stuff now! We will need to setup an HTTP server in order to do the TLS communication with clients. This is very important as we have a login page to the admin part of our application so we don't want to have our password not encrypted in the wire between the client and the server. 

We decided to use let's encrypt and Nginx to do that. Let's encrypt is a certificate authority which will verify that we are the owner of the domain and give us certificates. The certificates are a pair of keys which are just two files.

The first step is to generate the certificates on the local machine. We will use `certbot` to do that.

Then, we need to configure nginx to find the certificate to encrypt the ongoing and outgoing traffic.

```
worker_processes  1;

events {
	worker_connections  1024;
}


http {
	include       mime.types;
	default_type  application/octet-stream;

	sendfile        on;

	keepalive_timeout  65;


	upstream public {
		server web:3000;
	}

	server {
		listen 80;
		server_name #DOMAIN#;

		location ~ /.well-known {
			allow all;
			root /var/www/#DOMAIN#/;
		}

    location /favicon.ico {
      root /var/www/;
    }

    location / {
      return 301 https://$host$request_uri;
    }
	}

  server {
    listen 443 ssl;
    server_name #DOMAIN#;

    ssl_certificate /etc/letsencrypt/live/www.#DOMAIN#/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.#DOMAIN#/privkey.pem;

    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 10m;

    # ssl_dhparam /etc/ssl/dhparams/dhparam.pem;

    ssl_protocols TLSv1.1 TLSv1.2;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DES-CBC3-SHA:!ADH:!AECDH:!MD5;
    ssl_prefer_server_ciphers on;

    resolver 8.8.8.8 8.8.4.4;
    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";
		
    location ~ /.well-known {
			allow all;
			root /var/www/#DOMAIN#/;
		}
    
    location / {
      proxy_pass http://public/;
    }
  }


}
```

You can use the same config file and replace `#DOMAIN#` by your domain name.