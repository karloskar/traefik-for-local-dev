## Traefik for local development

Ports are nice. There are many of them. But remembering which port is used for which service
stops being fun whenever the number of projects increase.

Being able to reference a project by a subdomain to localhost (`service-name.localhost`) seemed
like a good idea. This is one solution to that idea using [Traefik](https://github.com/traefik/traefik).

### Prerequisite: create the shared network

The `localdevelopment` network is created once, by hand, and referenced here as
external:

```bash
docker network create --subnet 172.21.0.0/24 localdevelopment
```

Compose only deletes networks it owns, so `docker compose down` here now leaves
the network — and every other project attached to it — alone. Pick another
subnet if that range collides with something on your machine.

For extending the Traefik configuration the recommendation is to create a `compose.override.yaml`
file and make the necessary adjustments there.

A few machine-specific values are read from a `.env` file instead, so that
`compose.yaml` stays generic:

| variable       | default        | what it does                                       |
| -------------- | -------------- | -------------------------------------------------- |
| `BIND_IP`      | `0.0.0.0`      | interface the proxy publishes on, e.g. `127.0.0.1` |
| `HOST_GATEWAY` | `host-gateway` | what `host.docker.internal` resolves to            |

`BIND_IP` and `HOST_GATEWAY` live here rather than in the override because
Compose _appends_ `ports` and `extra_hosts` when merging — an override can add
an entry but never replace one, so both values have to be set in the base file.

### Rootless Docker

Under rootless Docker the `host-gateway` alias resolves to the gateway of the
container's own network namespace, which is not the host, so file-provider
services pointing at `host.docker.internal` will not connect. Set
`HOST_GATEWAY=10.0.2.2` (the slirp4netns gateway) in `.env` — this needs
`dockerd-rootless` to run _without_ `--disable-host-loopback`.

The daemon also serves its socket from `$XDG_RUNTIME_DIR` instead of
`/var/run`. Point at it from `compose.override.yaml`, keeping the container
path so it replaces the bind in `compose.yaml`:

```yaml
services:
  traefik:
    volumes:
      - /run/user/1000/docker.sock:/var/run/docker.sock
```

### Adding routers and services

Traefik is configured to use both the Docker and the File provider. Which let's you add a host
with Docker labels like this

```yaml
# a docker-compose example
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.service-1.rule=Host(\`service-1.localhost\`)"
```

or, for situations where you might have something running that is not containerized, by
adding a file in the dynamic directory (you need to create a directory called dynamic
in this directory). Maybe something like `dynamic/service-2.yml` with
content like this

```yaml
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

```yaml
# dynamic/tls.yml
tls:
  certificates:
    - certFile: "/etc/traefik/certs/{certificate-filename}.pem"
      keyFile: "/etc/traefik/certs/{key-filename}.pem"
```

and add

```yaml
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

### Docker network `localdevelopment`

Put your other services on this network and they can reach each other by
container name. Declare it as external there too, so no project but the creator
owns it:

```yaml
networks:
  default:
    external: true
    name: localdevelopment
```
