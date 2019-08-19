---
layout: post
title:  "HTTPS with Nginx and Let’s Encrypt"
author: "Nabil"
categories: DevOps
---

# Introduction

## What is HTTPS and why HTTPS?

HTTPS is HTTP over SSL or TLS. HTTP is a protocol that tells computers how to communicate. HTTPS is encrypted. The data is encrypted before being sent through the wire. Encryption is critical if you are sending credentials to connect to your Facebook account or if you are connecting to your online banking. It's very easy for anyone to sniff the network and see the data being sent. However, if the data is encrypted, the man in the middle will not understand the data being transfered.
Plus, Google has decided to down rank all websites which are not in HTTPS. 

## What is NGINX?

NGINX is a well established HTTP server. Basically, you download, edit a configuration file and run the server. We will configure it to intercept the HTTP queries and redirect the queries to our backend. Our NGINX server will be responsible for HTTPS as well.

# Architecture

Now that we understand what is NGINX and the motivation behing HTTPS, let's get to what we want to achieve in this tutorial. For that, lets look at this diagram:

![High level architecture diagram](/assets/nginx-https.png)

This is the architecture as we want it: We have on the left a desktop and a phone which are hitting our NGINX server which routes the requests to our application backends. The question that remains is: why do we need NGINX in this setup. First, it acts as a load balancer between the two backends that we have stood up. 

But most importantly, it acts as a tls termination server. What does it mean? It means that NGINX will be responsible to encrypt the data when responding and to decrypt the data when receiving it. We need to generate the certificates and configure NGINX to do the encryption and decryption of the data. 

# Step by step implementation

## Step 1: Use Nginx as basic reverse proxy

Let's make Nginx as a reverse proxy: it will receive the requests from the outside world on port 80, proxy it to our server on the same machine and send the request back to the client. I produced a live lab that you can interact with. We will use [this](https://github.com/shekhargulati/python-flask-docker-hello-world) application as an example. 

The code for this is step [here](https://github.com/nabilbouab/tech-blog-labs/tree/step-1). Let's disect the config, part by part. 

### event context
The first part is the event context. It is used to set global options that affect how Nginx handles connections at a general level.

```
events {
	worker_connections  1024;
}
```

We are configuring nginx to handle 1024 connections at max per worker.

### http context
The second part of the configuration is the http context. It defines how hginx will handle the HTTP or HTTPS connections.

```
http {
	include       mime.types;
	default_type  application/octet-stream;

	sendfile        on;

	keepalive_timeout  65;


	upstream public {
		server web:5000;
	}

	server {
		listen 80;
		server_name our-awesome-domain.fr;

    location / {
        proxy_pass http://public/;
    }
	}
}
```


The most important part of this configuration is the following:

```
	upstream public {
		server web:5000;
	}

	server {
		listen 80;
		server_name our-awesome-domain.fr;

    location / {
        proxy_pass http://public/;
    }
	}
```
We create an upstream which define a group of server that we later can reference in the `proxy_pass` directive. 

Then, in the `server` directive, we define a **virtual host** which will host our application. For those who don't know, a virtual host is a technique which consists in running multiple websites (for example www.company1.com and www.company2.com) on the same server. Virtual hosts can either be "by IP" in which case an IP address is given to each web server, or "by domain name" in which case several domain name are served from the same IP address.


## Step 2: Generate the certificates with certbot

In order to get a certificate, we need to ask a trusted certificate authority to check that we are the owner of a domain and then issue a certificate. We will use let's encrypt in order to do this. But first, let's explain how does the certificate generation work in order to understand the artichecture.

### Certificate generation process
It is very well explained in [this](https://letsencrypt.org/how-it-works/) documentation. But let me try to explain it myself. Basically, our goal is to have an automated way to generate a certificate from our server.

#### Why do we need certificates?
What does let's encrypt need in order to issue a certificate? It needs to be able to verify that the machine asking for the certificate owns the domain. Then, it will issue a certificate to tell our clients that the communication is with the right server and that it is encrypted. Why do we need that? Let's pretend you are client of HSBC: when you are on HSBC.co.uk, you want to be cryptographically sure that you are sending your credentials to hsbc.co.uk and not to some random server of a hacker who wants to steal them. So, when lets encrypt creates the certificate - which is nothing but a key pair that you are going to exchange with the client to encrypt data - they need to make sure that they issue them to the right server. Now that we understand the why, let's talk about the technical details of the how is domain validated by the certificate authority.

#### How does the domain validation work?
First, I would like to emphasize the fact that the generation of the certificates needs to be done on the servers that are pointed by your DNS record for the challenge to work. You will understand, trust me :D.

  1. We install an agent on the server. In this case it's certbot.
  2. The agent creates a key pair. It will be used in order to encrypt the traffic.
  3. The certificate authority submits a challenge to the agent and a nonce that the agent needs to sign with its private key in order to prove that it owns the key pair. The challenge consists in putting a file at a location and the authority needs to be able to request it back at www.domain.co.uk. The CA verifies the signature on the nonce, and it attempts to download the file from the web server and make sure it has the expected content.

In our case, the "agent" is certbot. Once we setup the DNS record to point to the right IP address, we can basically run one command like so `letsencrypt certonly -a webroot --webroot-path=/var/www/folder_where_challenge_will_be_put -d www.domain.co.uk`. We will then need to let nginx know that it need to serve the folder `/var/www/folder_where_challenge_will_be_put` in order for let's encrypt to retrieve it successfully and verify the domain.

In order to run this command, we need to get certbot installed in the nginx container, so we add this line
```
RUN apt-get install -y certbot python-certbot-nginx 
```

to the nginx Dockerfile. Then, we rebuild the image and run a container from it. At this point we installed the certbot agent. We now run the command in the container spawned from this image:
```bash
  $ docker ps
  $ docker exec -it <NGINX_CONTAINER_ID> bash
  $ letsencrypt certonly -a webroot --webroot-path=/var/www/folder_where_challenge_will_be_put -d www.domain.co.uk
```

If we run this command, the challenge will fail. As we said previously, the challenge consists in retrieving a file that is created on our machine. We need to change the nginx config to serve the folder where the file has been created, in our example it's `/var/www/folder_where_challenge_will_be_put/`. 
We need to add this part to the config file:

```
  location ~ /.well-known {
    allow all;
    root /var/www/folder_where_challenge_will_be_put/;
  }
```

If the last command succeeds, we generated the certificates which have been installed at `/etc/letsencrypt/live/www.domain.co.uk/`.

You can find the code of this step [here](https://github.com/nabilbouab/tech-blog-labs/tree/step-2)


## Step 3: Make HTTPS work

This last step will consist in two steps:
  * We first need to make a redirection from HTTP to HTTPS
  * We need to make Nginx do the encryption/decryption by providing it access to the generated certificates

We need to make an HTTP redirection to HTTPS. Let's try to understand this better in a more detailed diagram:

![SSL termination diagram](/assets/nginx-ssl-termination.png)

In order to do the ssl termination, Nginx needs to be able to access the certificates which will be used to encrypt and decrypt the data. We need to change the Nginx config to point it to the newly generated certificates and make the redirection that way:

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
		server_name our-awesome-domain.fr;

		location ~ /.well-known {
			allow all;
			root /var/www/our-awesome-domain.fr/;
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
    server_name our-awesome-domain.fr;

    ssl_certificate /etc/letsencrypt/live/www.our-awesome-domain.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.our-awesome-domain.fr/privkey.pem;

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
			root /var/www/our-awesome-domain.fr/;
		}
    
    location / {
      proxy_pass http://public/;
    }
  }
}
```

The redirection is done by returning a 301 to the browser and redirecting on https link. So the request will come to our Nginx server, which will return a 301 telling the browser to request whatever it requested but with https and we will then receive the new request on https. We created a new virtual host listening on 443 which is the HTTPS port.

In order for Nginx to be able to access the certificate, we need to mount a volume which contains the certificates. We modify the docker-compose.yml file in the `nginx` service, to add the volumes and we need to expose port 443 as well:

```
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

You now need to run the docker-compose application on your server with `docker-compose up` and go to your browser typing www.our-awesome-domain.fr. The domain should be secure.

# References

  * https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts

  * Thanks to my former colleague [Jeremy Rabasco](https://jrabasco.me/) to whom I stole some of his [blog config](https://github.com/jrabasco/LSBEngine) to reverse engineer how does all this work.

  * For the virtual hosts explanation, I found a great one [here](https://httpd.apache.org/docs/2.4/vhosts/)