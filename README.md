# tortainer
(WIP) general purpose tor container images with a shared base.

In the below examples, `podman` should be interchangable with `docker` without any other changes to the commands, with the exception of `podman generate ...` to create systemd services.

All images should be available for the following platforms: `linux/amd64` (x86_64), `linux/arm/v7` (ARM 32bit), `linux/arm64` (aarch64).

## torbase
Base tor image, just alpine with tor.
```bash
# pick an ORPort for your relay.
ORPORT=$[ (${RANDOM} % (65536-1024)) + 1024 ]
# keep long term data in a persistent container volume (*very* important for relay identity keys, etc)
podman volume create tor-datadir
# launch the relay
podman run \
  -d \
  --rm \
  --name tor-relay \
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
# keep long term data in a persistent container volume (important for guard context, descriptor caches, etc)
podman volume create tor-datadir
# launch the proxy, should be accessible as a SOCKS5 proxy at 127.0.0.1:9050 on the host.
podman run \
  -d \
  --rm \
  --name torproxy \
  -v tor-datadir:/var/lib/tor \
  -p 127.0.0.1:9050:9050 \
  ghcr.io/guest42069/torproxy:latest
# test the tor connection
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
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
# keep long term data in a persistent container volume (important for guard context, descriptor caches, etc)
podman volume create tor-datadir
# n.b. the argument following --bridge should be the obfs4 bridge line you obtained via https://bridges.torproject.org/ and it should be in quotes.
podman run \
  -d \
  --rm \
  --name obfs4-proxy \
  -p 127.0.0.1:9050:9050 \
  -v tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/obfs4-proxy:latest --bridge "obfs4 10.20.30.40:12345 3D7D7A39CCA78C7B0448AFA147EF4CC391564D03 cert=YvJSxrXcnXYZ+C9hsIr18bwsm5u5dtZG9DrLTo8CqY8mZlBjhXcUssJJ185mX+JCc/LSnQ iat-mode=0"
# test the tor connection
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
```

## obfs4-bridge
Basic tor bridge (server), obfs4 listening on port 443 is currently hardcoded. Changing the obfs4 port for 443 is possible but would require some knowledge of containers, see the `Containerfile` to crib the existing `ENTRYPOINT`. Can be overridden with your own `--entrypoint` argument to `podman run ...`, the `--servertransportlistenaddr` argument is where the changed should be made.
```bash
# pick a random ORPort value to make bridge detection more expensive for censors.
ORPORT=$[ (${RANDOM} % (65536-1024)) + 1024 ]
# keep long term data in a persistent container volume (*very* important for bridge identity keys, etc)
podman volume create tor-datadir
# start the bridge
podman run \
  -d \
  --rm \
  --name obfs4-bridge \
  -p 443:443 \
  -p ${ORPORT}:${ORPORT} \
  -v tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/obfs4-bridge:latest \
  --contactinfo myemail@mydomain.com \
  --orport ${ORPORT} \
  --nickname myrelay
# check the logs to ensure the bridge is published correctly, etc
podman logs -f 
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
# keep long term data in a persistent container volume (important for guard context, descriptor caches, etc)
podman volume create tor-datadir
# launch the proxy, should be accessible as a SOCKS5 proxy at 127.0.0.1:9050 on the host.
podman run \
  -d \
  --rm \
  --name snowflake-proxy \
  -p 127.0.0.1:9050:9050 \
  -v tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/snowflake-proxy:latest
# test the tor connection
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
```

## snowflake-standalone
A standalone snowflake proxy, this acts as an entrypoint for other users who intend to use a snowflake client to access Tor.
```bash
# host networking is prefered, to avoid another layer of NAT traversal: container <-(NAT)-> host, or worse: container <-(NAT)-> host <-(NAT)-> router
podman run \
  -d \
  --rm \
  --name snowflake \
  --network host \
  ghcr.io/guest42069/snowflake-standalone:latest
```

## arti
An in-development rust implementation of Tor, as a client. However it only binds to localhost so it's only suitable for use *within pods* _or_ *with host networking*.
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

## systemd service
Podman (not Docker, sadly) can automatically generate systemd service files that allow you to turn existing containers or pods into systemd managed services.
In this example, we'll create a systemd service for a `snowflake-standalone` container.
```bash
# host networking is prefered, to avoid another layer of NAT traversal: container <-(NAT)-> host, or worse: container <-(NAT)-> host <-(NAT)-> router
# --label allows us to set a label to take advantage of the 'podman auto-update' command which automatically pulls new images for any labeled containers.
podman run \
  -d \
  --rm \
  --name snowflake \
  --label "io.containers.autoupdate=registry" \
  --network host \
  ghcr.io/guest42069/snowflake-standalone:latest
# generate the service file for the container under /etc/systemd/system, referencing the container by the name we assigned.
(cd /etc/systemd/system;podman generate systemd --new --name --files snowflake)
# enable the systemd service to ensure it's automatically started by the OS on boot.
systemctl enable --now container-snowflake.service
# the container will now log to journald like other services, keeps service logs in with other systemd services.
journalctl -u container-snowflake.service
```
