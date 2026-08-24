# Docker — Simple Explanation

## 1. What is a container?

Think of your laptop as an apartment building.

A **VM (Virtual Machine)** is like giving each tenant their own separate building — full plumbing, electricity, everything duplicated from scratch. Heavy, slow to boot.

A **container** is like giving each tenant their own apartment in the same building — they share the building's core infrastructure (the OS kernel) but have their own locked door, furniture, and rules. Lightweight, starts in seconds.

**Why we need them:** the "it works on my machine" problem. A container packages your app + all its dependencies (libraries, config, runtime) into one unit that runs identically everywhere — your laptop, your teammate's laptop, AWS, anywhere.

| | VM | Container |
|---|---|---|
| Boots in | minutes | seconds |
| Size | GBs | MBs |
| Isolation | full OS | process-level (shares host kernel) |
| Runs on | Hypervisor | Docker Engine |

## 2. Docker architecture, simplified

```
You type command → Docker Client → Docker Daemon (dockerd) → does the work
                                          ↓
                                   pulls Images from
                                          ↓
                                    Docker Registry (Docker Hub)
                                          ↓
                                   runs Images as Containers
```

- **Client** — the `docker` command you type.
- **Daemon** — the background service that actually builds/runs things.
- **Image** — a frozen recipe/template (e.g. `ubuntu`, `nginx`).
- **Container** — a running instance of that image.
- **Registry** — an online store of images (Docker Hub).

## 3. Install on AWS Ubuntu (EC2 instance)

SSH into your Ubuntu EC2 instance, then:

```bash
# update packages
sudo apt update -y

# install docker (official quick script)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# add your user to docker group (so you don't need sudo every time)
sudo usermod -aG docker $USER
newgrp docker

# verify
docker --version
docker run hello-world
```

`hello-world` just prints a message confirming Docker can pull an image and run it — that's your "it works" test.

## 4. Practice commands, with plain-English meaning

### Run Nginx (a web server) and see it in the browser

```bash
docker run -d --name mynginx -p 8080:80 nginx
```
- `-d` — run in the background (detached)
- `--name mynginx` — give it a nickname instead of a random ID
- `-p 8080:80` — "when someone visits port 8080 on my machine, send them to port 80 inside the container"

On AWS: open port 8080 in the EC2 Security Group, then visit `http://<EC2-public-ip>:8080`.

### Run Ubuntu and explore it like a mini Linux box

```bash
docker run -it ubuntu bash
```
- `-it` — interactive terminal, so you get a shell prompt inside the container
- Try `ls`, `apt update`, `cat /etc/os-release` — you're "inside" a fresh Ubuntu
- Type `exit` to leave

### See what's running

```bash
docker ps       # only running containers
docker ps -a    # all containers, including stopped
```

### Check logs of a running container

```bash
docker logs mynginx
```

### Run a command inside an already-running container

```bash
docker exec -it mynginx bash
```
This "jumps into" the live nginx container so you can peek around while it's still serving traffic — without stopping it.

### Stop and remove

```bash
docker stop mynginx
docker rm mynginx
```

## Quick mental model to remember it all

- `docker run` = create + start
- `docker ps` = who's alive
- `docker exec` = "teleport inside" a running one
- `docker logs` = "what has it been saying"
- `docker stop` / `docker rm` = kill it and clean up
