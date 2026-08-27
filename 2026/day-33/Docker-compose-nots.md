# Docker Compose: Multi-Container Basics

Everything you manually typed on the volumes/networking day — `--network`, `-v`, `docker network create` — gets replaced by one YAML file. This is where it all comes together.

## Setup

Modern Docker already bundles Compose as a plugin — you use `docker compose` (no hyphen), not the old standalone `docker-compose`.

```bash
docker compose version
```
If it's missing:
```bash
sudo apt update
sudo apt install docker-compose-plugin -y
```

## Your first Compose file

```bash
mkdir compose-basics && cd compose-basics
nano docker-compose.yml
```

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

Reading it plainly:
- `services:` — the list of containers you want to run
- `web:` — the service's name, which *also* becomes its DNS hostname for other containers
- `image: nginx` — same as `docker run nginx`
- `ports: "8080:80"` — same as `-p 8080:80`

```bash
docker compose up      # start it, visit http://<EC2-public-ip>:8080
docker compose down    # stop it
```

Notice: no manual `docker network create`, no `--name` — Compose sets all of that up for you automatically.

## Two-container example: WordPress + MySQL

The classic case — an app that needs a database, connected automatically by name.

```yaml
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - dbdata:/var/lib/mysql

  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  dbdata:
```

The parts that matter most:

- **`WORDPRESS_DB_HOST: db`** — this is the magic. `db` is just the name of the other service above it. Compose automatically puts both services on the same private network with working DNS, so `wordpress` can reach `db` by name — no IP addresses anywhere.
- **`volumes: - dbdata:/var/lib/mysql`** (under `db`) — same idea as `-v dbdata:/var/lib/mysql`. This is where MySQL's actual data lives.
- **`volumes: dbdata:`** at the bottom — declares the named volume so Compose manages it, same as `docker volume create`.
- **`depends_on: - db`** — starts `db` before `wordpress`. Worth remembering: this only waits for the *container* to start, not for MySQL to actually be ready to accept connections — a common gotcha, usually fixed later with healthchecks.

```bash
docker compose up -d
```
Visit `http://<EC2-public-ip>:8080` and complete the WordPress setup — that data gets written into MySQL.

**Prove the data survives:**
```bash
docker compose down
docker compose up -d
```
Refresh the browser — your site and login are still there. `docker compose down` removes containers and the network, but **not** named volumes — so `dbdata` survives and MySQL's data survives with it.

If you actually want to wipe everything, data included:
```bash
docker compose down -v
```

## Commands you'll use constantly

```bash
docker compose up -d              # start everything, in the background
docker compose ps                 # list this project's running containers
docker compose logs                # logs from every service
docker compose logs wordpress      # logs from just one service
docker compose logs -f db          # follow logs live
docker compose stop               # pause containers, keep them around
docker compose start              # resume them
docker compose down               # remove containers + network (keeps volumes)
docker compose up -d --build       # rebuild images, then start
```

**`stop` vs `down`:** `stop` is like parking the car — fast to resume, nothing is torn down. `down` is like scrapping it and rebuilding next time — a clean slate, but slower. Use `stop` while actively developing, `down` when you want a full reset.

## Environment variables

**Inline** (what's in the examples above) works fine for quick tests, but for anything real, keep secrets out of the YAML with a `.env` file:

```bash
nano .env
```
```
DB_ROOT_PASSWORD=rootpass
DB_NAME=wordpress
DB_USER=wpuser
DB_PASSWORD=wppass
```

Then reference them in `docker-compose.yml` with `${VARNAME}`:
```yaml
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - dbdata:/var/lib/mysql

  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
    depends_on:
      - db

volumes:
  dbdata:
```

Compose automatically reads a file named exactly `.env` in the same folder — no flag needed. To sanity-check that your variables actually resolved correctly before running anything:
```bash
docker compose config
```
This prints your compose file with every `${VAR}` swapped for its real value — a quick way to catch typos in variable names.

## Quick reference

| Command | Purpose |
|---|---|
| `docker compose up -d` | start everything, detached |
| `docker compose down` | stop + remove containers & network |
| `docker compose down -v` | also remove named volumes (wipes data) |
| `docker compose stop` / `start` | pause / resume without removing |
| `docker compose ps` | list this project's containers |
| `docker compose logs -f <service>` | live logs for one service |
| `docker compose config` | preview resolved config, great for debugging `.env` |
| `docker compose up -d --build` | rebuild images, then start |

## Key things to remember

- Compose automates everything you'd otherwise do by hand — networks, volumes, naming.
- Service names *are* the DNS hostnames — `WORDPRESS_DB_HOST: db` works because `db` is a service name, nothing more.
- `down` ≠ `down -v` — know which one deletes your data.
- `.env` files keep secrets out of the committed YAML — standard practice on real teams.
