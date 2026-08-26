# Docker Volumes & Networking

Two things that separate toy Docker usage from real production setups: **making data survive** a container's death, and **letting containers talk to each other**.

## The problem: containers are disposable

Start a Postgres container and put some data in it:
```bash
docker run -d --name pgtest -e POSTGRES_PASSWORD=mypass postgres
docker exec -it pgtest psql -U postgres
```
```sql
CREATE TABLE test (id INT, name TEXT);
INSERT INTO test VALUES (1, 'hello');
\q
```

Now kill it and start a fresh one:
```bash
docker stop pgtest
docker rm pgtest

docker run -d --name pgtest2 -e POSTGRES_PASSWORD=mypass postgres
docker exec -it pgtest2 psql -U postgres -c "SELECT * FROM test;"
```
**Result: error, the table doesn't exist.** Your data is gone.

**Why:** a container's writable layer only exists as long as the container does. The moment you `docker rm` it, that layer is deleted for good. Containers are meant to be disposable — the moment you rely on data living inside one, you're doing it wrong.

## Named volumes — the real fix

A **named volume** is storage Docker manages for you, completely outside the container's lifecycle.

```bash
docker volume create pgdata
docker run -d --name pgvol -e POSTGRES_PASSWORD=mypass \
  -v pgdata:/var/lib/postgresql/data postgres
```
`-v pgdata:/var/lib/postgresql/data` mounts the volume at the exact path where Postgres stores its actual database files.

Add data, then destroy the container (not the volume):
```bash
docker exec -it pgvol psql -U postgres -c "CREATE TABLE test (id INT, name TEXT);"
docker exec -it pgvol psql -U postgres -c "INSERT INTO test VALUES (1, 'hello');"

docker stop pgvol
docker rm pgvol
```

Start a brand new container on the *same* volume:
```bash
docker run -d --name pgvol2 -e POSTGRES_PASSWORD=mypass \
  -v pgdata:/var/lib/postgresql/data postgres
docker exec -it pgvol2 psql -U postgres -c "SELECT * FROM test;"
```
**Your row is still there.** The container is disposable; the volume is the durable part.

Check where Docker actually keeps it on disk:
```bash
docker volume inspect pgdata
```

## Bind mounts — direct access to a host folder

```bash
mkdir ~/mysite
echo "<h1>Bind mount test</h1>" > ~/mysite/index.html

docker run -d --name bindtest -p 8080:80 -v ~/mysite:/usr/share/nginx/html nginx
```
Visit `http://<EC2-public-ip>:8080` — you'll see your HTML. Now edit the file directly on the host:
```bash
echo "<h1>Updated live!</h1>" > ~/mysite/index.html
```
Refresh the browser — updated instantly, no rebuild, no restart.

### Named volume vs bind mount

| | Named Volume | Bind Mount |
|---|---|---|
| Managed by | Docker | You (an exact host path) |
| Location | Docker's internal storage | Any folder you choose |
| Best for | Production data (databases) | Local development (live code editing) |
| Portability | Same across machines | Depends on that exact path existing |

**Rule of thumb:** volumes for data that needs to survive and move between machines (databases); bind mounts for your own source code while developing (instant reload, no rebuilding images).

## Default bridge network — no name resolution

```bash
docker run -dit --name c1 ubuntu bash
docker run -dit --name c2 ubuntu bash

docker exec c1 ping -c 2 c2
```
**Fails** — "unknown host." The default bridge network doesn't do DNS lookup by container name.

Try by IP instead:
```bash
docker inspect -f '{{.NetworkSettings.IPAddress}}' c2
docker exec c1 ping -c 2 <c2-ip>
```
**Works.** On the default bridge, containers can only reach each other by IP.

## Custom network — fixes the DNS problem

```bash
docker network create my-app-net

docker run -dit --name c3 --network my-app-net ubuntu bash
docker run -dit --name c4 --network my-app-net ubuntu bash

docker exec c3 ping -c 2 c4
```
**Works — by name this time.**

**Why:** Docker runs an internal DNS server, but it's only active on user-defined networks, not the legacy default bridge (kept around just for backward compatibility). Any custom network gets automatic name-based discovery — this is exactly how Docker Compose lets services find each other by name.

## Putting it together — a real multi-container pattern

This is the shape most real backend setups take before anyone reaches for Compose or Kubernetes:

```bash
# custom network
docker network create app-net

# database, with a volume, on that network
docker volume create dbdata
docker run -d --name db --network app-net \
  -e POSTGRES_PASSWORD=mypass \
  -v dbdata:/var/lib/postgresql/data \
  postgres

# app container on the same network
docker run -dit --name app --network app-net ubuntu bash

# check the app can reach the db by name
docker exec app bash -c "apt update && apt install -y iputils-ping && ping -c 2 db"
```
The app reaches the database using just the hostname `db` — no IP addresses anywhere. This is exactly why a `docker-compose.yml` or a Kubernetes service just says `DB_HOST=db` and it works, regardless of which machine or IP the container actually ends up on.

## Quick reference

| Concept | Command |
|---|---|
| Create a volume | `docker volume create name` |
| Attach a volume | `-v volumename:/path/in/container` |
| Bind mount | `-v /host/path:/container/path` |
| List volumes | `docker volume ls` |
| Inspect a volume | `docker volume inspect name` |
| List networks | `docker network ls` |
| Create a network | `docker network create name` |
| Run on a network | `--network name` |
| Inspect a network | `docker network inspect name` |

## Key takeaways

- **Volumes** = durable data that outlives the container, managed by Docker.
- **Bind mounts** = a direct link to a folder on your host, great for live development.
- **Default bridge** = containers can only find each other by IP, no name resolution.
- **Custom networks** = built-in DNS, containers reach each other by name — the foundation of how every multi-container app (and eventually Compose/Kubernetes) actually works.
