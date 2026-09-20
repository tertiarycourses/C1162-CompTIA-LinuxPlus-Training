# Lab 13 — Containers with Docker & Podman

This lab covers CompTIA Linux+ XK0-006 Objective 2.6 (Given a scenario, manage containers). You will install Docker and Podman, pull and run images, manage volumes and networks, build a tiny image from a Dockerfile, prune unused images, and contrast rootful vs rootless container engines.

Run all commands on the Killercoda Ubuntu Playground:
https://killercoda.com/playgrounds/scenario/ubuntu

---

## Step 1 — Install Docker and Podman

```bash
apt update -y
apt install -y docker.io podman
systemctl start docker
docker --version
podman --version
```

`docker.io` is Ubuntu's packaged Docker Engine and `podman` is Red Hat's daemonless, rootless-friendly alternative that uses the same CLI shape.

---

## Step 2 — Pull and run a container

```bash
docker pull nginx
docker images
docker run -d -p 8080:80 --name web nginx
docker ps
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

`-d` detaches, `-p host:container` publishes a port, and `--name` assigns a friendly identifier. Pulling first is optional — `run` will pull if needed.

---

## Step 3 — Inspect, exec, logs

```bash
docker logs web | head
docker exec -it web bash -c "nginx -v; ls /usr/share/nginx/html"
docker inspect web | head -40
docker stats --no-stream web
```

`exec` runs a new process inside a running container, `logs` streams stdout/stderr, and `inspect` returns a JSON view of config, mounts, network, and state.

---

## Step 4 — Volumes for persistent data

```bash
mkdir -p /data
echo "<h1>Lab 13</h1>" > /data/index.html
docker run -d -p 8081:80 -v /data:/usr/share/nginx/html:ro --name web2 nginx
curl -s http://localhost:8081
```

Bind mounts (`-v host:container`) expose host paths inside the container. Adding `:ro` makes the mount read-only — useful for static assets.

---

## Step 5 — Build a tiny image

```bash
mkdir -p /tmp/hello && cd /tmp/hello
cat > app.sh <<'EOF'
#!/bin/sh
echo "Hello from $(hostname) as $(id -un)"
EOF
chmod +x app.sh

cat > Dockerfile <<'EOF'
FROM alpine:3.20
RUN adduser -D appuser
COPY app.sh /usr/local/bin/app.sh
USER appuser
ENTRYPOINT ["/usr/local/bin/app.sh"]
CMD []
EOF

docker build -t hello:1 .
docker run --rm hello:1
```

`FROM` sets the base, `USER` drops root, `ENTRYPOINT` is the fixed command, and `CMD` is the default argument list (overridable at `docker run`).

---

## Step 6 — Networks and port mapping

```bash
docker network create labnet
docker run -d --network labnet --name api nginx
docker run --rm --network labnet alpine sh -c "apk add --no-cache curl >/dev/null && curl -s -o /dev/null -w '%{http_code}\n' http://api"
docker network inspect labnet | head -30
```

User-defined bridge networks (`docker network create`) provide built-in DNS so containers can address each other by name.

---

## Step 7 — Rootless Podman and privileged note

```bash
podman run --rm alpine echo "hello from podman"
podman ps -a
echo "Privileged containers (--privileged) disable most isolation: device access, capabilities, AppArmor."
echo "Prefer unprivileged + specific --cap-add over --privileged."
```

Podman can run without a daemon and, configured for a non-root user, runs containers in a rootless user namespace — a strong defense-in-depth posture.

---

## Step 8 — Cleanup

```bash
docker stop web web2 api 2>/dev/null
docker rm web web2 api 2>/dev/null
docker network rm labnet 2>/dev/null
docker rmi hello:1 nginx alpine:3.20 2>/dev/null
docker image prune -af
podman rmi alpine 2>/dev/null
rm -rf /tmp/hello /data
apt remove -y docker.io podman
```

`image prune -af` removes all dangling and unused images. Packages and lab files are uninstalled to leave a clean playground.

---

## What you learned
- Installing and using Docker and Podman to run, inspect, and exec containers
- Persisting data with bind-mount volumes and connecting containers via user-defined networks
- Building images from a Dockerfile and contrasting rootless Podman with privileged Docker

## Free tools used
- Killercoda Ubuntu Playground — https://killercoda.com/playgrounds/scenario/ubuntu
- Docker Engine — https://docs.docker.com/engine/
- Podman — https://podman.io/
- nginx & Alpine images — https://hub.docker.com/
