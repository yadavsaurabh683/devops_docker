# Day 2: Docker Networking and Storage — Deep Dive
## Theme: Where Most Engineers Are Completely Lost

**Goal:** By end of Day 2, you will understand exactly what happens at the network and storage
layer when containers communicate, and you will be able to diagnose networking issues without
guessing.

**Estimated time:** 5–6 hours

---

## Background: The Networking Mental Model

When Docker creates a container, it:
1. Creates a **veth pair** (virtual ethernet cable) — one end in the container, one on the host
2. Connects the host end to the **docker0 bridge** (a software switch)
3. Sets up **iptables NAT rules** for traffic to/from the outside world
4. Assigns an IP from Docker's internal subnet (default: 172.17.0.0/16)

When you publish a port (`-p 8080:80`), Docker adds an iptables DNAT rule:
- Traffic to host:8080 → forward to container-IP:80

Understanding this means you can diagnose ANY networking issue.

---

## Exercise 2.1 — Dissecting the Docker Network Stack

**What you'll learn:** What Docker physically creates on your host for networking.

**Instructions:**

1. List existing networks:
```bash
docker network ls
```

You'll see: `bridge` (default), `host`, `none`.

2. Inspect the default bridge:
```bash
docker network inspect bridge
```

Note the `Subnet` (172.17.0.0/16) and `Gateway` (172.17.0.1).

3. On your host, look at the bridge interface:
```bash
ip addr show docker0
ip link show type bridge
```

4. Run two containers on the default bridge:
```bash
docker run -d --name c1 alpine sleep 600
docker run -d --name c2 alpine sleep 600
```

5. Find their IPs:
```bash
docker inspect c1 --format '{{.NetworkSettings.IPAddress}}'
docker inspect c2 --format '{{.NetworkSettings.IPAddress}}'
```

6. Look at the veth pairs on the host:
```bash
ip link show type veth
```

You'll see pairs like `veth1a2b3c`. One end is on docker0, the other is inside the container.

7. Try to ping c2 from c1:
```bash
docker exec c1 ping -c 3 <c2-IP>
```

Works, because they're on the same bridge.

8. Try to ping c2 from c1 **by name**:
```bash
docker exec c1 ping -c 3 c2
```

**This FAILS on the default bridge.** DNS resolution between containers only works on
**user-defined** networks. This is one of the most common Docker networking mistakes.

---

## Exercise 2.2 — User-Defined Networks and DNS

**What you'll learn:** Why user-defined networks are always preferred, and how Docker DNS works.

**Instructions:**

1. Create a custom network:
```bash
docker network create mynet
```

2. Start containers on this network:
```bash
docker run -d --name web --network mynet nginx:alpine
docker run -d --name client --network mynet alpine sleep 600
```

3. Ping by name from client:
```bash
docker exec client ping -c 3 web
```

**It works.** Docker's embedded DNS server (runs at 127.0.0.11 inside containers) resolves
service names to container IPs on user-defined networks.

4. Verify the DNS server inside the container:
```bash
docker exec client cat /etc/resolv.conf
```

You'll see `nameserver 127.0.0.11`. This is Docker's internal DNS.

5. Do a manual DNS lookup:
```bash
docker exec client nslookup web
```

6. Now connect c1 (from the previous exercise, on the default bridge) to mynet:
```bash
docker network connect mynet c1
docker exec c1 ping -c 3 web
```

A container can be on **multiple networks simultaneously**. Its routing table has routes for
each network.

> **Hint:** If you want to see what DNS names are resolvable inside a container, run
> `docker exec <container> cat /etc/hosts` — Docker injects static entries for known containers.

---

## Exercise 2.3 — Port Publishing and iptables: What Actually Happens

**What you'll learn:** The real mechanism behind `-p` port mapping.

**Instructions:**

1. Run a container with port published:
```bash
docker run -d --name webserver -p 8888:80 nginx:alpine
```

2. On the host, inspect iptables NAT rules:
```bash
sudo iptables -t nat -L DOCKER -n --line-numbers
# OR for newer systems using nftables:
sudo nft list ruleset | grep -A5 docker
```

Look for a DNAT rule pointing to the container's IP on port 80.

3. Get the container's IP:
```bash
docker inspect webserver --format '{{.NetworkSettings.IPAddress}}'
```

4. Access the container directly via its IP (without port mapping):
```bash
curl http://<container-IP>:80
```

Works from the host because the host can route to docker0's subnet.

5. Now test the mapped port:
```bash
curl http://localhost:8888
```

6. Stop the container and verify the iptables rule disappears:
```bash
docker stop webserver
sudo iptables -t nat -L DOCKER -n
```

**The rule is gone.** Docker manages iptables rules dynamically based on container lifecycle.

> **Production insight:** Docker modifies iptables rules in the `DOCKER` chain, which runs
> **before** your custom firewall rules. This means even if you block port 8888 with ufw,
> Docker's direct iptables manipulation bypasses it. This is a known security issue in
> production environments.

---

## Exercise 2.4 — Container Networking Failure Diagnosis

**Scenario (real-world type):** Two services should talk to each other. They can't. Debug it.

**Setup:**
```bash
# Create two networks to simulate an architecture mistake
docker network create frontend-net
docker network create backend-net

# App server on both networks (intentional)
docker run -d --name appserver \
  --network frontend-net \
  nginx:alpine

# DB server only on backend network
docker run -d --name dbserver \
  --network backend-net \
  alpine sleep 600
```

**Your task:** Figure out why `dbserver` cannot reach `appserver`, and fix it with the minimum
change required.

**Diagnosis steps (figure these out yourself before reading hints):**

```bash
# Step 1: Check what network each container is on
docker inspect appserver --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool
docker inspect dbserver --format '{{json .NetworkSettings.Networks}}' | python3 -m json.tool

# Step 2: Try to ping from dbserver
docker exec dbserver ping -c 2 appserver
# Will fail — different network, no DNS visibility

# Step 3: Fix by connecting appserver to backend-net
docker network connect backend-net appserver

# Step 4: Try again
docker exec dbserver ping -c 2 appserver
```

> **Hint:** Trying to debug connectivity? Always check: (1) are both containers on the same
> network? (2) are you using the right container name for DNS? (3) use `docker exec <c> wget -O-
> http://<target>:<port>` to test HTTP-level reachability, not just ping.

---

## Exercise 2.5 — Storage: Named Volumes vs Bind Mounts vs tmpfs

**What you'll learn:** When to use each storage type and what happens to data on container removal.

**Part A — Anonymous volumes (the trap):**

```bash
# Run postgres with no explicit volume
docker run -d --name pg1 \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# Write some data
docker exec -it pg1 psql -U postgres -c "CREATE DATABASE testdb;"
docker exec -it pg1 psql -U postgres -c "\l"

# Remove the container
docker rm -f pg1

# Check volumes
docker volume ls
```

An anonymous volume was created. It still exists after container removal, but it has an
unreadable hash name. This is the "data trap" — your data exists but is nearly unmanageable.

**Part B — Named volumes (correct approach):**

```bash
# Run with named volume
docker run -d --name pg2 \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

# Write data
docker exec -it pg2 psql -U postgres -c "CREATE DATABASE myapp;"

# Remove container
docker rm -f pg2

# Volume still exists and is named
docker volume ls | grep pgdata

# Start new container with same volume — data persists
docker run -d --name pg3 \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

docker exec -it pg3 psql -U postgres -c "\l"
```

Database `myapp` is still there. **Named volumes persist across container lifecycles.**

**Part C — Bind mounts (host directory):**

```bash
mkdir -p /tmp/myconfig
echo "server { listen 80; return 200 'Hello World'; add_header Content-Type text/plain; }" \
  > /tmp/myconfig/default.conf

docker run -d --name webconf \
  -p 9090:80 \
  -v /tmp/myconfig:/etc/nginx/conf.d:ro \
  nginx:alpine

curl localhost:9090
```

Now change the config file on the host and reload nginx — no container rebuild needed:
```bash
echo "server { listen 80; return 200 'Updated Config'; add_header Content-Type text/plain; }" \
  > /tmp/myconfig/default.conf
docker exec webconf nginx -s reload
curl localhost:9090
```

**Part D — tmpfs (ephemeral memory storage):**

```bash
docker run -d --name tmptest \
  --tmpfs /tmp:size=64m \
  alpine sleep 600

# Write to tmpfs
docker exec tmptest sh -c 'dd if=/dev/zero of=/tmp/testfile bs=1M count=10'
docker exec tmptest ls -lh /tmp/testfile
```

tmpfs is in memory. When the container stops, the data is gone. Use for: session files,
scratch space, secrets that must not touch disk.

**Questions to answer:**
> 1. What happens to a named volume when you run `docker rm`? What about `docker rm -v`?
> 2. If two containers mount the same named volume simultaneously, what are the risks?
> 3. When would you choose a bind mount over a named volume in production?

---

## Exercise 2.6 — Networking Simulation: Multi-Tier App

**Scenario:** Simulate a 3-tier application (frontend, API, database) with proper network
segmentation. The database must NOT be reachable from the frontend network.

**Architecture:**
```
[frontend-net] <-> [frontend container] <-> [api container] <-> [db container] <-> [backend-net]
```

API container is on BOTH networks. DB is only on backend. Frontend cannot talk to DB.

```bash
# Create networks
docker network create app-frontend
docker network create app-backend

# Database (backend only)
docker run -d --name db \
  --network app-backend \
  -e POSTGRES_PASSWORD=secret \
  postgres:16-alpine

# API (bridges both networks)
docker run -d --name api \
  --network app-backend \
  nginx:alpine

docker network connect app-frontend api

# Frontend (frontend only)
docker run -d --name frontend \
  --network app-frontend \
  nginx:alpine
```

**Verify segmentation:**
```bash
# Frontend can reach API
docker exec frontend ping -c 2 api

# Frontend CANNOT reach db (different network, not connected)
docker exec frontend ping -c 2 db   # Should fail with: bad address 'db'

# API can reach db
docker exec api ping -c 2 db
```

> **Hint:** If `ping` isn't available in the container, use
> `docker exec <container> wget -q --spider http://<target>:<port>` to test TCP connectivity.
> Or install it: `docker exec <container> apk add --no-cache iputils`

---

## Day 2 Checkpoint

Answer these without looking:

1. What is a veth pair and why does Docker create one per container?
2. Why does DNS resolution by container name fail on the default bridge network?
3. Where does Docker's embedded DNS server listen inside a container?
4. What iptables chain does Docker use for port publishing? Why does this bypass ufw?
5. What is the difference between a named volume and an anonymous volume?
6. When a container is removed with `docker rm`, what happens to its named volume?
7. What storage driver does Docker use by default on Linux? (Check `docker info`)

---

## Day 2 Summary

| Concept | What You Proved |
|---|---|
| Docker networking internals | Saw veth pairs, docker0 bridge, iptables NAT rules |
| User-defined network DNS | Container name resolution only on custom networks |
| Port publishing mechanism | iptables DNAT rule created/destroyed with container |
| Named vs anonymous volumes | Data persistence behavior across container lifecycle |
| Bind mounts for config | Hot-reload nginx config without container rebuild |
| Network segmentation | 3-tier app with isolated DB network |
