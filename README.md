# tortainer
(WIP) general purpose tor container images with a shared base.

In the below examples, `podman` should be interchangable with `docker` without any other changes to the commands.

All images should be available for the following platforms: `linux/amd64` (x86_64), `linux/arm/v7` (ARM 32bit), `linux/arm64` (aarch64).

## torbase
Base tor image, just alpine with tor.
```bash
ORPORT=$[ (${RANDOM} % (65536-1024)) + 1024 ]
podman volume create tor-datadir
podman run \
  -v tor-datadir:/var/lib/tor \
  -p ${ORPORT}:${ORPORT} \
  ghcr.io/guest42069/torbase:latest \
  --orport ${ORPORT} \
  --nickname myrelay \
  --contactinfo myemail@mydomain.com
```

## torproxy
Basic tor (client) proxy. **N.B.** binds to 0.0.0.0 inside the container, to allow for use in inter-container networking or exposure by publishing, do *not* use with host networking.
```bash
podman volume create tor-datadir
podman run \
  -v tor-datadir:/var/lib/tor \
  -p 127.0.0.1:9050:9050 \
  ghcr.io/guest42069/torproxy:latest
```

## obfs4
Just an obfs4 binary, located inside a scratch image at `/obfs4proxy`. Used for later images, not much use on it's own.

```Dockerfile
FROM ghcr.io/guest42069/obfs4:latest AS bin
FROM docker.io/library/alpine:latest
COPY --from=bin /obfs4proxy /usr/bin/obfs4proxy
...
```

## obfs4-proxy
Basic tor (client) proxy configured to use an obfs4 bridge. See notes for `torproxy` above.
```bash
podman volume create tor-datadir
podman run -p 127.0.0.1:9050:9050 -v tor-datadir:/var/lib/tor ghcr.io/guest42069/obfs4-proxy:latest --bridge "obfs4 10.20.30.40:12345 3D7D7A39CCA78C7B0448AFA147EF4CC391564D03 cert=YvJSxrXcnXYZ+C9hsIr18bwsm5u5dtZG9DrLTo8CqY8mZlBjhXcUssJJ185mX+JCc/LSnQ iat-mode=0"
```

## obfs4-bridge
Basic tor bridge (server), obfs4 listening on port 443.
```bash
ORPORT=$[ (${RANDOM} % (65536-1024)) + 1024 ]
podman volume create tor-datadir
podman run \
  -p 443:443 \
  -p ${ORPORT}:${ORPORT} \
  -v tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/obfs4-bridge:latest \
  --contactinfo myemail@mydomain.com \
  --orport ${ORPORT} \
  --nickname myrelay
```
## snowflake
Just a snowflake client binary, located inside a scratch image at `/client`. Used for later images, not much use on it's own.

```Dockerfile
FROM ghcr.io/guest42069/snowflake:latest AS bin
FROM docker.io/library/alpine:latest
COPY --from=bin /client /usr/bin/client
...
```

## snowflake-proxy
Basic tor (client) proxy configured to use a snowflake bridge. See notes for `torproxy` above.
```bash
podman volume create tor-datadir
podman run \
  -p 127.0.0.1:9050:9050 \
  -v tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/snowflake-proxy:latest
```

## snowflake-standalone
A standalone snowflake proxy, this acts as an entrypoint for other users who intend to use a snowflake client to access Tor.
```bash
podman run --network host ghcr.io/guest42069/snowflake-standalone:latest
```

## arti
An in-development rust implementation of Tor, as a client. However it only binds to localhost so it's only suitable for use within pods or with host networking.
```bash
# host networking
podman run --detach --network host ghcr.io/guest42069/arti:latest
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
# pods (where the containers will share localhost)
podman pod create --name arti
podman run --detach --pod arti --rm ghcr.io/guest42069/arti:latest
podman run --pod arti --rm -it docker.io/library/alpine sh
# the alpine container we are now in can now access arti's socksport on 127.0.0.1:9050
apk add curl
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
```
