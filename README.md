## Traefik for local development

Ports are nice. There are many of them. But remembering which port is used for which service
stops being fun whenever the number of projects increase.

Being able to reference a project by a subdomain to localhost (`service-name.localhost`) seemed
like a good idea. This is one solution to that idea using [Traefik](https://github.com/traefik/traefik).

### Adding routers and services

Traefik is configured to use both the Docker and the File provider. Which let's you add a host
with Docker labels like this

```
# a docker-compose example
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.service-1.rule=Host(\`service-1.localhost\`)"
```

or, for situations where you might have something running that is not containerized, by
adding a file in the dynamic directory (you need to create a directory called dynamic
in this directory). Maybe something like `dynamic/service-2.yml` with
content like this

```
http:
  services:
    service-2:
      loadBalancer:
        servers:
        - url: "http://host.docker.internal:{PORT_OF_RUNNING_SERVICE}/"
  routers:
    service-2:
      entryPoints:
        - "http"
      rule: "Host(`service-2.localhost`)"
      # reference the created service
      service: "service-2"
```

If everything was setup correctly, both `service-1.localhost` and `service-2.localhost`
should now be good to go.


### TLS

For encrypted connections, create a directory called `local-certs`. Then use mkcert or a
similar tool to get your certificates.

Put a file in the dynamic directory like so:

```
# dynamic/tls.yml
tls:
  certificates:
    - certFile: "/etc/traefik/certs/{certificate-filename}.pem"
      keyFile: "/etc/traefik/certs/{key-filename}.pem"
```

and add

```
http:
  routers:
    # [ ... ]
    service-2-tls:
      entryPoints:
        - "https"
      rule: "Host(`service-2.localhost`)"
      service: "service-2"
      tls: {}
```

if needed or use Traefik labels for a Docker container.
