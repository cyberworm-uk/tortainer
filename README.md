# tortainer
General purpose tor container images with a shared base.

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
# _OR_ pods (where the containers will share localhost)
podman pod create --name arti
podman run --detach --pod arti --rm ghcr.io/guest42069/arti:latest
podman run --pod arti --rm -it docker.io/library/alpine sh
# the alpine container we are now in can now access arti's socksport on 127.0.0.1:9050
apk add curl
(curl -x socks5h://127.0.0.1:9050/ https://check.torproject.org | grep -F Congratulations.) && echo "Success" || echo "Failure"
# To persist state, etc mount a volume to /arti inside the container
podman volume create arti-state
podman run --detach --pod arti --volume arti-state:/arti --rm ghcr.io/guest42069/arti:latest
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
podman run --detach --pod arti --volume arti-config:/arti/.config/arti/arti.d --rm ghcr.io/guest42069/arti:latest
```

## onionshare
[`onionshare-cli`](https://onionshare.org/) is a project used to provide web, file sharing and receiving and chat services over Tor onion services.

This is still a bit of a work in progress and may need some tweaking in future and more extensive documentation of use cases.

```bash
# create storage to container persistent files
podman create volume onionshare-config
# launch a chat server (I.E. requires only the .onion address and not an additional private key, )
podman run --rm -v onionshare-config:/home/onionshare/.config/onionshare ghcr.io/guest42069/onionshare --chat
#... outputs the following
#Give this address and private key to the recipient:
#http://yud4dyoi4pk344rjjlm34bgeszflbra2qvxc732kxvssgt2muv6vzryd.onion
#Private key: 6MZ7PXM7GO3H2QNMOM6VMXGUE7PJ2KJWUJE2TETVGU35YCCDHTEA
#... ctrl-c
```

In the above example, the servers private key won't persist (I.E. relaunching it won't give the same address). We can create a persistent config (stored in the volume) with the `--persistent ...` flag, which is a path to a config file.

```bash
podman run --rm -v onionshare-config:/home/onionshare/.config/onionshare ghcr.io/guest42069/onionshare --chat --persistent persistent/test
#... outputs the following
#Give this address and private key to the recipient:
#http://vrxgl5w4b5g3eoo4ibrpaejlyieeofgeanrjllmrru3ikt6v6tis3xad.onion
#Private key: V3GL36LUBLKS3L55SIW6327JCGZRZ7CQ5SZ6G3QZYNXK4S3POSCA
#... ctrl-c
podman run --rm -v onionshare-config:/home/onionshare/.config/onionshare ghcr.io/guest42069/onionshare --chat --persistent persistent/test
#... outputs the following
#Give this address and private key to the recipient:
#http://vrxgl5w4b5g3eoo4ibrpaejlyieeofgeanrjllmrru3ikt6v6tis3xad.onion
#Private key: V3GL36LUBLKS3L55SIW6327JCGZRZ7CQ5SZ6G3QZYNXK4S3POSCA
#... ctrl-c
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

## hidden / onion service website
This example is `podman` specific (as we'll be taking advantage of pods, which aren't directly interchangable with `docker`).
In this example creates a static hello world page using nginx.
This could be further hardened and a more sophisticated example may be added later. This example only intends to outline the general process and isn't intended to be production ready.
```bash
# create persistent storage for onion webservice config
podman volume create onion-nginx-conf
podman volume create onion-var-www-html
# create persistent storage for onion tor data
podman volume create onion-tor-datadir
# create a pod to run our services in
podman pod create --name onion
# create the tor container within to handle running the onion service
podman run \
  -d \
  --rm \
  --pod onion \
  --label "io.containers.autoupdate=registry" \
  --name onion-tor \
  -v onion-tor-datadir:/var/lib/tor \
  ghcr.io/guest42069/torbase:latest \
  --socksport 0 \
  --hiddenservicedir /var/lib/tor/website \
  --hiddenserviceport "80 127.0.0.1:80"
# create a basic nginx config
echo 'server {
  listen 80;
  access_log off;
  root /var/www/html;
  index index.html;
  server_name _;
  server_tokens off;
}' > $(podman volume inspect -f '{{ .Mountpoint }}' onion-nginx-conf)/website.conf
# create a basic html page
echo '<html>
<head>
<title>Hello World</title>
</head>
<body>
<h1>Hello World</h1>
</body>
</html>' > $(podman volume inspect -f '{{ .Mountpoint }}' onion-var-www-html)/index.html
# create the nginx container within the pod to handle the incoming http requests
podman run \
  -d \
  --rm \
  --pod onion \
  --label "io.containers.autoupdate=registry" \
  --name onion-nginx \
  -v onion-nginx-conf:/etc/nginx/conf.d \
  -v onion-var-www-html:/var/www/html \
  docker.io/library/nginx:alpine
# obtain our onion hostname
cat $(podman volume inspect -f '{{ .Mountpoint }}' onion-tor-datadir)/website/hostname
# visit the site with Tor Browser and check that you've got your hello world message.
# podman log onion-tor to check tor logs
# podman log onion-nginx to check nginx logs
# if all looks well, generate a systemd service for the onion pod.
(cd /etc/systemd/system; podman generate systemd --new --name --files onion)
# enable the systemd service for the pod.
systemctl enable --now pod-onion.service
# there will also be services for the two containers, these are set as required by the onion pod service and will start with it automatically.
# journalctl -u container-onion-tor.service to check tor logs
# journalctl -u container-onion-nginx.service to check nginx logs
```
