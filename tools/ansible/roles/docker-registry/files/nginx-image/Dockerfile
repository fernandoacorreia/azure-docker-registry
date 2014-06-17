# A nginx frontend for a Docker registry server.

FROM phusion/baseimage:0.9.10

MAINTAINER Fernando Correia <fernandoacorreia@gmail.com>

RUN DEBIAN_FRONTEND=noninteractive apt-get -qq update
RUN DEBIAN_FRONTEND=noninteractive apt-get -qqy install nginx
ADD nginx.conf /etc/nginx/nginx.conf
ADD http-443.conf /etc/nginx/conf.d/http-443.conf
ADD http-80.conf /etc/nginx/conf.d/http-80.conf
RUN mkdir /etc/service/nginx
ADD nginx-run /etc/service/nginx/run
ADD nginx-container.pub /tmp/nginx-container.pub
RUN cat /tmp/nginx-container.pub >> /root/.ssh/authorized_keys && rm -f /tmp/nginx-container.pub
mkdir -p /etc/nginx/secret
ADD docker-registry.htpasswd /etc/nginx/secret/docker-registry.htpasswd
ADD ssl.crt /etc/nginx/secret/ssl.crt
ADD ssl.key /etc/nginx/secret/ssl.key
RUN chown -R www-data: /etc/nginx/secret && chmod 0600 /etc/nginx/secret/*
