# tortainer
General purpose tor container images with a shared base.

In the below examples, `podman` should be interchangable with `docker` without any other changes to the commands, with the exception of `podman generate ...` to create systemd services.

All images should be available for the following platforms: `linux/amd64` (x86_64), `linux/386` (i386), `linux/arm/v7` (arm), `linux/arm64` (aarch64).

## Container Image Documentation

- [torbase](#torbase) an image which serves as a common base that more complete images are build on.
- [torproxy](#torproxy) an image for a regular Tor proxy (client) image.
- [obfs4](#obfs4) an image which contains compiled obfs4-proxy binaries only.
- [lyrebird-proxy](#lyrebird-proxy) an image for a Tor proxy (client) connecting over meek_lite, obfs2, obfs3, obfs4 and scramblesuit.
- [obfs4-bridge](#obfs4-bridge) an image for a Tor bridge (server) allowing clients to connect into the Tor network over obfs4.
- [snowflake](#snowflake) an image which contains compiled snowflake client binary only.
- [snowflake-proxy](#snowflake-proxy) an image for a Tor proxy (client) connecting over snowflake.
- [snowflake-standalone](#snowflake-standalone) an image for a snowflake entrypoint, serves as a go between for snowflake clients and snowflake bridges.
- [arti](#arti) an image for an in-development, experimental Tor implementation in Rust.
- [onion service](#onion-service) example of a basic onion service running with containers.

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
  ghcr.io/cyberworm-uk/torbase:latest \
  --orport ${ORPORT} \
  --nickname myrelay \
  --contactinfo myemail@mydomain.com
```
or, as a quadlet

`tor-relay.container`
```.service
[Unit]
Description=Tor relay container

[Container]
Image=ghcr.io/cyberworm-uk/torbase:latest
AutoUpdate=registry
Exec=--orport 9001 --nickname myrelay --contactinfo myemail@mydomain.com

Volume=tor-datadir.volume:/var/lib/tor
PublishPort=9001:9001
PublishPort=[::]:9001:9001

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

`tor-datadir.volume`
```.service
[Volume]
User=100
Group=65533
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
  ghcr.io/cyberworm-uk/torproxy:latest
# test the tor connection
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
```

## obfs4
Just an obfs4 binary, located inside a scratch image at `/obfs4proxy`. Used for later images, not much use on it's own.

```Dockerfile
FROM ghcr.io/cyberworm-uk/obfs4:latest AS bin
FROM docker.io/library/alpine:latest
COPY --from=bin /obfs4proxy /usr/bin/obfs4proxy
...
```

## lyrebird-proxy
Basic tor (client) proxy configured to use an meek_lite, obfs2, obfs3, scramblesuit or obfs4 bridge. See notes for `torproxy` above.
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
  ghcr.io/cyberworm-uk/obfs4-proxy:latest --bridge "obfs4 10.20.30.40:12345 3D7D7A39CCA78C7B0448AFA147EF4CC391564D03 cert=YvJSxrXcnXYZ+C9hsIr18bwsm5u5dtZG9DrLTo8CqY8mZlBjhXcUssJJ185mX+JCc/LSnQ iat-mode=0"
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
  ghcr.io/cyberworm-uk/obfs4-bridge:latest \
  --contactinfo myemail@mydomain.com \
  --orport ${ORPORT} \
  --nickname myrelay
# check the logs to ensure the bridge is published correctly, etc
podman logs -f 
```
## snowflake
Just a snowflake client binary, located inside a scratch image at `/client`. Used for later images, not much use on it's own.

```Dockerfile
FROM ghcr.io/cyberworm-uk/snowflake:latest AS bin
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
  ghcr.io/cyberworm-uk/snowflake-proxy:latest
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
  ghcr.io/cyberworm-uk/snowflake-standalone:latest
```

## arti
An in-development, experimental rust implementation of Tor.
```bash
# host networking
podman run --detach -p 127.0.0.1:9050:9050 ghcr.io/cyberworm-uk/arti:latest
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
# _OR_ pods (where the containers will share localhost)
podman network create arti
podman run --detach --network arti --name arti --rm ghcr.io/cyberworm-uk/arti:latest
podman run --network arti --rm -it docker.io/library/alpine sh
# the alpine container we are now in can now access arti's socksport on 127.0.0.1:9050
apk add curl
(curl -x socks5h://arti:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
# To persist state, etc mount a volume to /arti inside the container
podman volume create arti-state
podman run --detach --network arti --volume arti-state:/arti --name arti --rm ghcr.io/cyberworm-uk/arti:latest
# To use a custom arti.toml config file place it in a directory and mount it into the container.
# For example, to configure a bridge ...
podman volume create arti-config
echo '
[bridges]

enabled = true
bridges = [
  # These are just examples, and will not work!
  "Bridge 192.0.2.66:443 8C00000DFE0046ABCDFAD191144399CB520C29E8",
  "Bridge 192.0.2.78:9001 6078000DFE0046ABCDFAD191144399CB52FFFFF8",
]
' > $(podman volume inspect -f '{{ .Mountpoint }}' arti-config)/bridges.toml
podman run --detach --network arti --volume arti-config:/arti/.config/arti/arti.d --name arti --rm ghcr.io/cyberworm-uk/arti:latest
```

## onion service
This example is `podman` specific, using quadlets.

*N.B.* While this uses internal networking to try to minimise nginx making external connections, DNS resolution is likely still going to work from nginx. As such you should be careful to take appropriate precautions to ensure that your outbound DNS is anonymized or entirely blocked for the containers.

`onion-tor.container`
```.service
[Unit]
Description=Onion Tor container

[Container]
Image=ghcr.io/cyberworm-uk/torproxy:latest
AutoUpdate=registry

Exec=--hiddenservicedir /var/lib/tor/onion --hiddenserviceport "80 systemd-onion-nginx:80"

Volume=onion-tor.volume:/var/lib/tor

Network=onion.network
Network=onion-ext.network

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

`onion-tor.volume`
```.service
[Volume]
User=100
Group=65533
```

`onion.network`
```.service
[Network]
IPv6=true
Internal=true
```

`onion-ext.network`
```.service
[Network]
IPv6=true
```

Here is the core of the Onion Service, we have a Tor container along with storage for it's data (onion keys, guards, etc). Note that we have two networks, one internal and the other not. We will attach our service to the internal one so it cannot directly access the internet.

`onion-nginx.container`
```.service
[Unit]
Description=Onion Nginx container

[Container]
Image=docker.io/library/nginx:alpine
AutoUpdate=registry

Volume=onion-nginx-config.volume:/etc/nginx/conf.d
Volume=onion-nginx-content.volume:/usr/share/nginx/html

Network=onion.network

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
RequiredBy=onion-tor.service
```

`onion-nginx-config.volume`
```.service
[Volume]
```

`onion-nginx-content.volume`
```.service
[Volume]
```

A very basic nginx, with a volume setup for config files and site content. Note that it's only attached to the internal network. Also note it has a `RequiredBy` for the `onion-tor.service` which is the tor container, this is to ensure that Tor can resolve the provided hostname of `systemd-onion-nginx` for the `HiddenServicePort` directive.

The below assumes these will be run as rootless containers, with the .container, .network and .volume unit files being placed in a folder at `~/.config/containers/systemd/`. We'll start the services.
```bash
systemctl --user daemon-reload
systemctl --user start onion-tor
```

We can obtain the address of the onion service that was created by running the following
```bash
podman exec systemd-onion-tor cat /var/lib/tor/onion/hostname
```
