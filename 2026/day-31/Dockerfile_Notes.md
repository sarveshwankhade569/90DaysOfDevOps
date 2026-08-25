# Dockerfile — Building Your Own Images

This is the day you stop just *using* Docker images and start *building* them.

## Your first Dockerfile

```dockerfile
FROM ubuntu
RUN apt update && apt install -y curl
CMD ["echo", "Hello from my custom image!"]
```

- `FROM ubuntu` — every image needs a starting point. This one starts from a clean Ubuntu filesystem.
- `RUN ...` — runs *during the build*, baking curl permanently into the image. Chaining `update && install` in one line is a normal habit (more on why below).
- `CMD [...]` — the default thing to do when someone runs a container from this image, unless they say otherwise.

Build and run it:
```bash
docker build -t my-ubuntu:v1 .
docker run my-ubuntu:v1
```
`-t my-ubuntu:v1` tags the image with a name and version. The trailing `.` tells Docker "look in this current folder for the files you need."

## A more complete example

```dockerfile
FROM ubuntu
RUN apt update && apt install -y python3
WORKDIR /app
COPY index.html .
EXPOSE 8000
CMD ["python3", "-m", "http.server", "8000"]
```

| Line | What it actually does |
|---|---|
| `FROM ubuntu` | Base OS layer |
| `RUN apt update && apt install -y python3` | Bakes python3 into the image at build time |
| `WORKDIR /app` | Creates `/app` and makes it the "current folder" for everything after — like `mkdir /app && cd /app` |
| `COPY index.html .` | Copies the file from your machine into `/app` inside the image |
| `EXPOSE 8000` | Documentation only — tells people "this listens on 8000." Doesn't actually open the port. |
| `CMD [...]` | Starts a simple Python web server on port 8000 by default |

```bash
docker build -t my-full-app:v1 .
docker run -d -p 8000:8000 --name fullapp my-full-app:v1
```

⚠️ **Common mistake:** `EXPOSE` doesn't open anything by itself. You still need `-p` at `docker run`, and you still need to open the port in your EC2 Security Group.

## CMD vs ENTRYPOINT

This trips almost everyone up early on — so here's the difference proven with two tiny tests.

**CMD** is just a default — easy to override:
```dockerfile
FROM ubuntu
CMD ["echo", "hello"]
```
```bash
docker run cmd-test                  # → hello
docker run cmd-test echo "overridden"  # → overridden
```
Whatever you type after the image name completely replaces `CMD`.

**ENTRYPOINT** is fixed — it always runs, no matter what:
```dockerfile
FROM ubuntu
ENTRYPOINT ["echo"]
```
```bash
docker run entry-test hello            # → hello
docker run entry-test "overridden text"  # → overridden text
```
Anything you pass gets *appended* to it, not replacing it.

**Rule of thumb:**
- Use `CMD` for a sensible default the user can freely override.
- Use `ENTRYPOINT` when the image has one job it should always do.
- Best of both: `ENTRYPOINT ["python3", "app.py"]` + `CMD ["--port", "8000"]` — the entrypoint is fixed, the CMD gives default arguments that can still be overridden.

## Building on top of someone else's image

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
```
No `CMD` needed — `nginx:alpine` already has its own built-in entrypoint/CMD that starts nginx. You're just extending a well-built image instead of starting from scratch. This is the pattern you'll use constantly in real projects.

```bash
docker build -t my-website:v1 .
docker run -d -p 8080:80 --name mysite my-website:v1
```

## .dockerignore

Your build context (`.`) gets sent to Docker entirely before building — including `.git`, `node_modules`, or secrets in `.env`. That's slow and risky. Exclude them:

```
node_modules
.git
*.md
.env
```

Works exactly like `.gitignore`, just for Docker builds.

## Layer caching — why instruction order matters

Docker builds top to bottom and **caches each layer**. If a layer's instruction hasn't changed, Docker reuses the cached result instead of redoing it — but the moment one layer changes, that layer *and everything after it* gets rebuilt.

**Bad order** (what beginners do):
```dockerfile
FROM ubuntu
COPY . .                                   # your code — changes constantly
RUN apt update && apt install -y python3   # rarely changes
```
Every code edit invalidates the cache and reinstalls python3 from scratch. Slow.

**Good order:**
```dockerfile
FROM ubuntu
RUN apt update && apt install -y python3   # rarely changes — put it early
COPY . .                                    # changes constantly — put it last
```
Now editing your code only invalidates the `COPY` layer onward — the expensive install stays cached.

**Rule to remember:** order your Dockerfile from *least frequently changing* at the top to *most frequently changing* at the bottom. Dependencies first, source code last.

## Quick reference

| Instruction | Purpose |
|---|---|
| `FROM` | base image |
| `RUN` | executes at build time, bakes into the image |
| `CMD` | default command, overridable at `docker run` |
| `ENTRYPOINT` | fixed command, `docker run` args get appended |
| `COPY` | copy files from host into the image |
| `WORKDIR` | set the working directory inside the image |
| `EXPOSE` | documentation only, doesn't publish ports |
| `.dockerignore` | excludes files from the build context |
