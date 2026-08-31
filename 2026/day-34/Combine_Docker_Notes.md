# Docker Notes — Day 34 to Day 37

## Day 34 – Real-World Multi-Container Apps

A production-like stack usually looks like this — a web app, a database, and a cache, all talking to each other by service name:

```yaml
services:
  web:
    build: ./app
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: mypass
    volumes:
      - dbdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  cache:
    image: redis

volumes:
  dbdata:
```

### `depends_on` alone isn't enough

Plain `depends_on: - db` only waits for the **container to start** — not for Postgres to actually be ready to accept connections. That gap causes a classic bug: your app starts, tries to connect, and fails because the database process is still booting up.

**The fix is a healthcheck** — a command Docker runs periodically to check if a service is *actually* ready:
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 5s
  timeout: 5s
  retries: 5
```
Then make your app wait for that healthy status specifically:
```yaml
depends_on:
  db:
    condition: service_healthy
```
Now `web` only starts once `db` reports healthy, not just "started."

### Restart policies

```yaml
restart: always        # always restart, no matter why it stopped — even after a reboot
restart: on-failure    # only restart if it exited with an error (non-zero exit code)
```

**When to use which:** `always` is right for things that should basically never be down — a database, a core service. `on-failure` fits things where a clean, deliberate stop (`docker stop`) should stay stopped, but a crash should trigger a retry.

### Building your app from a Dockerfile instead of a pre-built image

```yaml
services:
  web:
    build: ./app
```
Instead of pulling someone else's image, Compose builds yours from `./app/Dockerfile`. After changing your code:
```bash
docker compose up -d --build
```
One command rebuilds the image and restarts the container with your changes.

### Named networks, volumes, and labels

```yaml
services:
  web:
    build: ./app
    networks:
      - appnet
    labels:
      - "project=my-app"

networks:
  appnet:

volumes:
  dbdata:
```
Defining these explicitly (instead of relying on Compose's defaults) makes larger projects easier to reason about, and labels help you organize/filter containers later (`docker ps --filter label=project=my-app`).

### Scaling — and why it breaks with plain port mapping

```bash
docker compose up --scale web=3
```
This tries to start 3 copies of the `web` service. **If your compose file has `ports: "5000:5000"` on that service, this fails** — all 3 containers try to bind the same host port 5000, and only one can hold it.

**Why:** a fixed host-port mapping assumes exactly one container. To actually scale, you'd drop the fixed host port and put a load balancer (like nginx or Traefik) in front, letting Docker assign internal ports dynamically. This is exactly the kind of problem tools like Kubernetes exist to solve properly.

---

## Day 35 – Multi-Stage Builds & Docker Hub

### The problem: single-stage builds are bloated

A typical single-stage Dockerfile for a compiled app:
```dockerfile
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```
This image ships with the *entire Go compiler and toolchain* inside it — hundreds of MBs — just to run one small binary. Nobody needs the compiler at runtime.

### The fix: multi-stage builds

```dockerfile
# Stage 1: build
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

# Stage 2: run
FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
```

**What's happening:**
- `AS builder` names the first stage so you can reference it later.
- `COPY --from=builder /app/server /server` grabs *just the compiled binary* from the first stage and drops it into a brand-new, minimal image — the whole Go toolchain gets left behind.

**Why the final image is so much smaller:** the build tools, source code, and intermediate files never make it into the final image — only the one artifact you actually need to run does. This is the standard way real teams ship production images: build once with all the heavy tools, but ship only the result.

Compare sizes to see it for yourself:
```bash
docker build -t myapp:single -f Dockerfile.single .
docker build -t myapp:multistage -f Dockerfile.multistage .
docker images | grep myapp
```

### Pushing to Docker Hub

```bash
docker login
docker tag myapp:latest yourusername/myapp:v1
docker push yourusername/myapp:v1
```

- `docker tag` doesn't create a new image — it just gives an existing one an additional name, in the `username/repo:tag` format Docker Hub expects.
- Once pushed, anyone (including you, on a different machine) can pull it:
```bash
docker pull yourusername/myapp:v1
```

**`:latest` vs a specific tag:** `latest` is just a regular tag, not automatically "the newest version" — it's whatever was last pushed without a specific tag. In real projects, pull a specific version tag (`v1`, `v2.3.1`) so you always know exactly what you're running; relying on `latest` in production is a common source of "it worked yesterday" bugs.

### Image best practices

```dockerfile
FROM node:20-alpine        # minimal base image, specific version — not `latest`

RUN addgroup app && adduser -S -G app app   # create a non-root user
USER app                                     # run as that user, not root

WORKDIR /app
COPY package*.json ./
RUN npm install --production   # dependencies rarely change — cache this layer
COPY . .                        # code changes often — copy it last

CMD ["node", "server.js"]
```

Four habits worth building:
- **Minimal base image** (`alpine` over `ubuntu`) — smaller, smaller attack surface.
- **Non-root user** — if the app is compromised, it isn't running as root inside the container.
- **Combine `RUN` commands** with `&&` where it makes sense — fewer layers, smaller image.
- **Pin base image versions** — `node:20-alpine`, not `node:latest` — so builds are reproducible.

---

## Day 36 – Dockerizing a Full Application

This is the day it all comes together — taking a real app from zero to a working, shareable Docker setup.

### The shape of a full project

```
myapp/
├── app/                  # application source code
│   └── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env
└── README.md
```

### The Dockerfile — combining everything learned so far

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup app && adduser -S -G app app
USER app
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
This uses everything from the last two days: a multi-stage build, a minimal base image, and a non-root user.

### The Compose file — app + database, wired together properly

```yaml
services:
  app:
    build: ./app
    ports:
      - "3000:3000"
    environment:
      DB_HOST: db
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - appnet

  db:
    image: postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - appnet

networks:
  appnet:

volumes:
  dbdata:
```

Everything here is a concept from the last few days, just applied together: a custom network, a named volume for persistence, `.env` for config, and a healthcheck gating startup order.

### Shipping it

```bash
docker tag myapp:latest yourusername/myapp:v1
docker push yourusername/myapp:v1
```

### The real test: does it work from scratch?

```bash
docker compose down -v
docker system prune -a -f
docker compose up -d
```
If it comes up clean using nothing but your `docker-compose.yml`, `.env`, and the pushed image — it's genuinely done. If something breaks here, it usually means a hardcoded local path, a missing environment variable, or an assumption that only held true on your machine. Fix it here, not later.

### README worth including in the project

A short README should cover:
- What the app does
- How to run it: `docker compose up -d`
- Which environment variables are needed (list them, don't need to give values)
- The Docker Hub link to the pushed image

---

## Day 37 – Revision & Cheat Sheet

### Quick-fire answers worth knowing cold

- **Image vs container** — image is the frozen template; container is a running instance of it.
- **Data inside a removed container** — gone. Only volumes (or bind mounts) survive.
- **How containers on a custom network talk** — by service/container name, via Docker's built-in DNS.
- **`docker compose down` vs `down -v`** — the plain version removes containers + network but keeps named volumes; `-v` also deletes the volumes (and your data with them).
- **Why multi-stage builds matter** — the final image only contains the runtime artifact, not the build tools used to create it — much smaller and more secure.
- **`COPY` vs `ADD`** — `COPY` just copies files, plain and predictable. `ADD` can also auto-extract tar archives and fetch remote URLs — more "magic," which is exactly why most style guides say to prefer `COPY` unless you specifically need what `ADD` does.
- **`-p 8080:80`** — map port 8080 on the host to port 80 inside the container; visiting the host on 8080 reaches the container's port 80.
- **Checking Docker's disk usage** — `docker system df`.

### Docker Cheat Sheet

**Containers**
```bash
docker run -d --name x image      # run, detached
docker run -it image bash         # run, interactive shell
docker ps                         # running containers
docker ps -a                      # all containers
docker exec -it x bash            # jump into a running container
docker logs -f x                  # follow logs
docker stop x                     # graceful stop
docker rm x                       # remove
```

**Images**
```bash
docker build -t name:tag .        # build from Dockerfile
docker pull name:tag              # download
docker push user/name:tag         # upload to Docker Hub
docker tag local:tag user/repo:tag  # rename/retag
docker images                     # list local images
docker rmi name                   # remove an image
```

**Volumes**
```bash
docker volume create name
docker volume ls
docker volume inspect name
docker volume rm name
```

**Networks**
```bash
docker network create name
docker network ls
docker network inspect name
docker network connect name container
```

**Compose**
```bash
docker compose up -d              # start, detached
docker compose down               # stop + remove containers/network
docker compose down -v            # ...and remove volumes too
docker compose ps                 # list this project's containers
docker compose logs -f service    # follow one service's logs
docker compose up -d --build      # rebuild then start
```

**Cleanup**
```bash
docker system df                  # disk usage
docker system prune -a -f         # remove everything unused
```

**Dockerfile instructions**
```dockerfile
FROM        # base image
RUN         # run a command at build time
COPY        # copy files from host into image
WORKDIR     # set working directory inside image
EXPOSE      # documents the port (doesn't publish it)
CMD         # default command, overridable at `docker run`
ENTRYPOINT  # fixed command, run args get appended
```
