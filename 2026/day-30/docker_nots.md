# Docker Images & Container Lifecycle — Simple Explanation

## 1. Images vs Containers — the core relationship

- **Image** = a recipe/blueprint. Frozen, read-only.
- **Container** = a cake baked from that recipe. Running, has its own state.

You can bake many cakes (containers) from one recipe (image). Deleting a cake doesn't delete the recipe — you can always bake another.

## 2. Pull some images and compare sizes

```bash
docker pull nginx
docker pull ubuntu
docker pull alpine

docker images
```

You'll see something like:
```
REPOSITORY   TAG       SIZE
nginx        latest    ~190MB
ubuntu       latest    ~78MB
alpine       latest    ~7MB
```

**Why is alpine so much smaller?** Ubuntu ships a full-featured OS — lots of tools, docs, and libraries pre-installed. Alpine is a stripped-down Linux built specifically for containers: minimal shell, minimal libraries (`musl` instead of `glibc`), no extra packages. Great for production because it's small and fast — but don't be surprised if tools you assume exist (like `curl` or `bash`) aren't there by default, and you need to install them yourself.

To look at an image's metadata (layers, env vars, exposed ports, entrypoint, architecture):
```bash
docker inspect nginx
```

To remove an image you don't need:
```bash
docker rmi alpine
```

## 3. Image layers

```bash
docker image history nginx
```

Each row is one **layer**, roughly matching one instruction in the Dockerfile that built it (`RUN`, `COPY`, `ADD`, etc.).

- A layer with a size next to it → it added files (installed a package, copied in code).
- A layer showing `0B` → it's metadata-only (things like `CMD`, `EXPOSE`, `ENV` — they configure the container but don't add any files).

**Why layers matter:** Docker caches each one. If you rebuild an image and only your last couple of lines changed, Docker reuses all the earlier cached layers instead of redoing the whole build — much faster. It also means if two images share the same base (like `ubuntu`), that shared layer is only stored once on disk, not duplicated for each image.

## 4. The full container lifecycle

Walk through this once and the whole lifecycle clicks:

```bash
# 1. Create without starting — the container exists but isn't running yet
docker create --name mybox ubuntu sleep 1000
docker ps -a   # status: Created

# 2. Start it
docker start mybox
docker ps -a   # status: Up

# 3. Pause it — freezes all its processes in place
docker pause mybox
docker ps -a   # status: Paused

# 4. Unpause
docker unpause mybox
docker ps -a   # status: Up again

# 5. Stop it — a graceful shutdown (sends SIGTERM, gives it a chance to clean up)
docker stop mybox
docker ps -a   # status: Exited

# 6. Restart it
docker restart mybox
docker ps -a   # status: Up

# 7. Kill it — forceful, immediate (sends SIGKILL, no time to clean up)
docker kill mybox
docker ps -a   # status: Exited

# 8. Remove it entirely
docker rm mybox
docker ps -a   # gone completely
```

**Simple mental model:**
`create` = build the box → `start` = flip it on → `pause` = freeze it → `stop` = ask it politely to shut down → `kill` = pull the plug → `rm` = throw the box away.

## 5. Working with a live Nginx container

```bash
# run it in the background
docker run -d --name webserver -p 8080:80 nginx

# see its logs so far
docker logs webserver

# follow the logs live, like `tail -f`
docker logs -f webserver
# Ctrl+C just stops you watching — the container keeps running

# jump inside and look around
docker exec -it webserver bash
ls /usr/share/nginx/html
exit

# or run one command without fully entering
docker exec webserver cat /etc/nginx/nginx.conf

# inspect it — IP address, ports, mounts, everything
docker inspect webserver
docker inspect -f '{{.NetworkSettings.IPAddress}}' webserver   # just the IP
```

On your AWS Ubuntu EC2 instance, visit `http://<EC2-public-ip>:8080` in your browser — just make sure port 8080 is open in the Security Group first.

## 6. Cleaning up

```bash
docker stop $(docker ps -q)        # stop every running container at once
docker container prune -f          # remove all stopped containers
docker image prune -f              # remove unused (dangling) images
docker system df                   # see how much disk Docker is using
docker system prune -a -f          # nuclear option: wipe everything unused
```

## Quick recap

| Command | What it does |
|---|---|
| `docker create` | build a container, don't run it yet |
| `docker start` / `stop` / `restart` | polite lifecycle control |
| `docker pause` / `unpause` | freeze / unfreeze |
| `docker kill` | force stop, no cleanup time |
| `docker rm` / `rmi` | delete a container / an image |
| `docker logs -f` | live log stream |
| `docker exec` | run a command inside, or jump into, a live container |
| `docker inspect` | full JSON details about a container or image |
| `docker system prune` | clean up unused disk space |
