+++
title = 'Traefik'
date = '2025-08-31T18:20:42+07:00'
draft = false
tags = [ "traefik", "docker"]
categories = [ "devops" ]
+++

Traefik Configuration Starter

This is basic traefik configuration that often I use when using traefik. This
configuration focusing on using file providers as traefik configuration therefore
the `docker-compose.yaml` looks clean. But as for the downside, you need to copy
multiple files like the `docker-compose.yaml` and `traefik.yaml` as base
configuration. Then the dynamic configuration files need to be created based
on your needs.

This `docker-compose.yaml` file will create 2 containers; traefik and whoami
service to test if this configuration working as we want.

```docker-compose.yaml
services:
  traefik:
    image: traefik:latest
    restart: always
    environment:
      - "TZ=Asia/Jakarta"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    volumes:
      - ./traefik.yaml:/etc/traefik/traefik.yaml
      - ./dynamic:/etc/traefik/dynamic
      - ./letsencrypt:/etc/traefik/letsencrypt
    healthcheck:
      test: ["CMD", "traefik", "healthcheck"]
      interval: 60s
      retries: 3

  whoami:
    image: traefik/whoami
    restart: always
```

This `traefik.yaml` file is the base configuration for traefik.
It already includes HTTP Redirection to HTTPS, SSL Automatic
Renewal, and automatic configuration discovery.
Additional notes, if you want the dynamic configuration to be
applied automatically, it should be referenced as a file inside
`providers.file.directory`. If you add new directory inside it and
put the configuration inside the child directory, then the automatic
reload will not work.

```traefik.yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

api:
  dashboard: true
  insecure: false
  debug: false

log:
  format: json
  level: INFO

accessLog:
  addInternals: true
  format: json
  filters:
    statusCodes:
      - "200"
      - "400-404"
      - "500-503"
    retryAttempts: true
    minDuration: "10ms"

certificatesResolvers:
  letsencrypt:
    acme:
      email: "test@example.com"
      storage: "/etc/traefik/letsencrypt/acme.json"
      httpChallenge:
        entryPoint: web

providers:
  file:
    directory: "/etc/traefik/dynamic"
    watch: true
```

This is a basic dynamic configuration to route all request
to domain `example.com` to be received by `service-whoami`

```dynamic/svc-a.yaml
http:
  routers:
    router-whoami:
      rule: "Host(`example.com`)"
      service: service-whoami

  services:
    service-whoami:
      loadBalancer:
        servers:
          - url: http://whoami:80
```
