---
layout: post
title:  "HTTPS with Nginx and Let’s Encrypt"
author: "Nabil"
categories: DevOps
---

# Introduction

## What is HTTPS and why HTTPS?

HTTPS is secure communication protocol that encrypts the data during the transit. 
The first motivation to setup HTTPS, is obviously security: Encryption is critical when sending credentials to connect to Facebook account or online banking. Tools like wireshark enables anyone to sniff the network and analyse the data in transit.
The second motivation is about Search engine optimisation: Google has decided to down rank all websites which are not in HTTPS.

## What is NGINX?

NGINX is a well established HTTP server which will receive the HTTP requests and will send HTTP responses. We will configure NGINX to encrypt the data when responding and decrypt the data when receiving requests.

# Architecture

Now that we understand the motivation behing HTTPS, and what is NGINX, let me present the high level architecture of what we are going to build throughout this tutorial.

![High level architecture diagram](/assets/nginx-https.png)

We have on the left a desktop and a phone which are hitting our NGINX server. NGINX routes the requests to our application backends which are running on the same machine. In other words, we want NGINX to act as a **reverse proxy**.

# Step by step implementation

## Step 1: Use Nginx as basic reverse proxy

As a first step, we setup NGINX as an HTTP reverse proxy, we will kick the HTTPS part down the road. Basically, at the end of this step our NGINX server will be able to receive requests on port 80, proxy it to our server on the same machine and send the request back to the client. We will use [this](https://github.com/shekhargulati/python-flask-docker-hello-world) application as an example that we want to serve with https. 

The code for this is step [here](https://github.com/nabilbouab/tech-blog-labs/tree/step-1). The most important part of this configuration is the following:

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

### Certificate generation process

#### Why do we need certificates?
What does let's encrypt need in order to issue a certificate? It needs to be able to verify that the machine asking for the certificate owns the domain. Then, it will issue a certificate to tell our clients that the communication is with the right server and that it is encrypted. Why do we need that? Let's pretend you are client of HSBC: when you are on HSBC.co.uk, you want to be cryptographically sure that you are sending your credentials to hsbc.co.uk and not to some random server of a hacker who wants to steal them. So, when lets encrypt creates the certificate - which is nothing but a key pair that you are going to exchange with the client to encrypt data - they need to make sure that they issue them to the right server. Now that we understand the why, let's talk about the technical details of the how is domain validated by the certificate authority.

#### How to generate the certificate?
In order to issue a certificate, a trusted authority like **let's encrypt** needs to check that we are the owner of a domain. This process is very well explained in [this](https://letsencrypt.org/how-it-works/) document. Let's encrypt enable us to automate the generation of a certificate from our server.

The certificate generation process is the following:

  1. We install an agent on the server. In this case it's certbot.
  2. The agent creates a key pair. It will be used in order to encrypt the traffic.
  3. The certificate authority submits a challenge to the agent and a nonce that the agent needs to sign with its private key in order to prove that it owns the key pair. The challenge consists in putting a file at a location and the authority needs to be able to request it back at www.domain.co.uk. The CA verifies the signature on the nonce, and it attempts to download the file from the web server and make sure it has the expected content.

In our case, the "agent" is [certbot](https://certbot.eff.org/). Once we setup the DNS record to point to the right IP address, we can basically run one command like so `letsencrypt certonly -a webroot --webroot-path=/var/www/folder_where_challenge_will_be_put -d www.domain.co.uk`. We will then need to let nginx know that it need to serve the folder `/var/www/folder_where_challenge_will_be_put` in order for let's encrypt to retrieve it successfully and verify the domain. _I would like to note that the generation of the certificates needs to be done on the servers that are pointed by your DNS record for the challenge to work._

In order to run this command, we need to get certbot installed in the nginx container, so we add this line to the Dockerfile
```
RUN apt-get install -y certbot python-certbot-nginx 
```

Then, we rebuild the image and run a container from it. At this point we installed the certbot agent. We now run the command in the container spawned from this image:
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

This last step will consist in two sub steps:
  * We first need to make a redirection from HTTP to HTTPS
  * We need to make NGINX do the encryption/decryption by providing it access to the generated certificates

![SSL termination diagram](/assets/nginx-ssl-termination.png)

In order to do the ssl termination, NGINX needs to be able to access the certificates which will be used to encrypt and decrypt the data. We need to change the NGINX config to point it to the newly generated certificates and make the redirection that way:

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

In order for NGINX to be able to access the certificate, we need to mount a volume which contains the certificates. We modify the docker-compose.yml file in the `nginx` service, to add the volumes and we need to expose port 443 as well:

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


The redirection is done by returning a 301 to the browser and redirecting the same url to an https link. It would work this way: the request will come to our NGINX server, which will return a 301 telling the browser to request whatever it requested but with https and we will then receive the new request on https on port 443. This is why, we created a new virtual host listening on 443.

You now need to run the docker-compose application on your server with `docker-compose up` and go to your browser typing www.our-awesome-domain.fr. The domain should be secure.

# References

  * https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts

  * Thanks to my former colleague [Jeremy Rabasco](https://jrabasco.me/) to whom I stole some of his [blog config](https://github.com/jrabasco/LSBEngine) to reverse engineer how does all this work.

  * For the virtual hosts explanation, I found a great one [here](https://httpd.apache.org/docs/2.4/vhosts/)