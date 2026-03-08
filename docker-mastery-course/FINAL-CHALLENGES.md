# Final Challenges
## Complex Questions to Test Your Docker Mastery

These questions are designed to be answered **without running Docker** — they test conceptual
depth, diagnostic reasoning, and production-level thinking. If you can answer all of them
confidently, you have genuinely mastered Docker fundamentals.

Some questions require multi-part answers. Some require you to trace through multiple layers
of the system (process → namespace → kernel → network). That is intentional.

---

## Category 1: Core Concepts & Internals

**Q1.**
A developer says: "My Docker container crashed. I'll just restart it and it'll be fine."
What is the operational risk in this statement? What evidence is destroyed by restarting?
What should the engineer do instead, and in what order?

---

**Q2.**
You run `docker run -d --name myapp nginx:alpine` and then on the host you run `ps aux | grep nginx`.
You see nginx processes with PIDs like 12847, 12891, 12902.
Inside the container, `ps aux` shows PIDs 1, 5, 6.

Explain:
- Why are the PIDs different?
- Which Linux kernel feature causes this?
- If you run `kill 12847` from the host, what happens to the container?
- What if you run `kill 1` from inside the container?

---

**Q3.**
You have a container running with `--memory=256m`.
The application inside grows past 256 MB.

Trace the exact sequence of events that occurs from the moment the memory limit is exceeded
to the moment you see the container listed as `Exited` in `docker ps -a`.
Which kernel subsystem is involved? How do you confirm the cause of death?

---

**Q4.**
Explain OverlayFS using this scenario:
- You pull `python:3.12-slim` — 5 layers
- You build a custom image on top — adds 3 more layers
- You run 4 containers from this image simultaneously

How many copies of the base layers exist on disk?
When container 2 writes a file to `/app/data.txt`, does container 3 see it?
When container 2 is removed, what happens to `/app/data.txt`?
What if `/app/data.txt` was in a named volume instead?

---

## Category 2: Networking

**Q5.**
You run:
```bash
docker run -d --name svc -p 8080:80 nginx:alpine
```

Trace the entire network path when a request arrives at `localhost:8080` on the host
until it reaches the nginx process inside the container.

Name every component involved: kernel interface, iptables chain, Docker network bridge,
veth pair, container network namespace.

---

**Q6.**
You have two containers on the **default bridge network**:
```bash
docker run -d --name alpha alpine sleep 600
docker run -d --name beta alpine sleep 600
```

`alpha` can ping `beta` by IP. But `docker exec alpha ping beta` fails with "bad address."
`beta` can ping `alpha` by IP too.

Explain precisely WHY name-based ping fails.
What is the single minimal change that fixes this?
After the fix, where does the DNS query from `alpha` go, and what resolves it?

---

**Q7.**
A colleague claims: "I added `ufw deny 9000` to block port 9000 on the server, but my
Docker container that publishes port 9000 is still accessible from outside."

Why? Explain the iptables evaluation order and exactly where Docker's rules sit relative
to UFW. How would you actually block external access to a Docker-published port?

---

**Q8.**
You run:
```bash
docker run -d --name app1 --network net1 myapp:v1
docker run -d --name app2 --network net2 myapp:v1
docker network connect net2 app1
```

Draw the network topology. Can `app2` reach `app1` by name now? Can `app1` reach `app2`
by name? What IP addresses does `app1` have? How does routing work for each network?

---

## Category 3: Image Building & Dockerfile

**Q9.**
Given this Dockerfile:
```dockerfile
FROM python:3.12-slim
COPY . .
RUN pip install -r requirements.txt
COPY config.json .
CMD python app.py
```

You change one line in `app.py` and rebuild. Which layers are rebuilt from scratch?
Rewrite the Dockerfile to maximize cache efficiency without changing its functional output.

---

**Q10.**
A teammate writes this Dockerfile:
```dockerfile
FROM node:20
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
RUN npm install --production
RUN rm .npmrc
CMD ["node", "server.js"]
```

They argue: "I delete `.npmrc` in the last RUN so the token is gone."
Are they right? How would you prove them wrong with a single Docker command?
Provide the correct solution that genuinely prevents the token from persisting.

---

**Q11.**
Explain multi-stage builds by answering:
- In a 3-stage Dockerfile (stages: deps, builder, runtime), what gets shipped in the final image?
- If you use `COPY --from=builder /app/binary /binary` in the runtime stage, and the
  builder stage had 15 intermediate layers, how many of those layers appear in the final image?
- What is the `scratch` base image? Can you run `docker exec` into a scratch container?
  How would you debug one in production?

---

**Q12.**
You have two images:
- `myapp:v1` — 800 MB, based on `python:3.12`, runs as root, no health check
- `myapp:v2` — 120 MB, based on `python:3.12-slim`, runs as `appuser`, has health check

Beyond size, name 5 concrete operational or security differences between these two images
and explain the real-world implication of each.

---

## Category 4: Docker Compose & Multi-Service

**Q13.**
```yaml
services:
  api:
    image: myapi:latest
    depends_on:
      - db
  db:
    image: postgres:16-alpine
```

Your team reports that the API crashes on startup 30% of the time in CI.
The remaining 70% it works fine. No code changes were made.

Diagnose the root cause. What is the fix? Write the corrected YAML snippet.

---

**Q14.**
You have a Compose stack: `nginx → api → db`.
You run `docker compose kill db` to simulate a DB outage.

Answer:
- What HTTP status code does nginx return to clients? Why?
- Does the `api` container exit? Why or why not?
- When you bring `db` back with `docker compose start db`, what happens to `api`?
  Does it automatically start working again, or must it restart? What determines this?
- How would you design the stack to automatically restore without any container restart?

---

**Q15.**
A Compose file has:
```yaml
services:
  worker:
    image: myworker:latest
    restart: always
    command: python job.py
```

`job.py` reads from a queue, processes one job, and exits with code 0.
After 1000 successful jobs, your monitoring shows the container has restarted 1000 times.
Each restart adds a 2-second cold start overhead — 33 minutes of wasted time.

What is the problem? What is the correct restart policy and why?
How would you redesign the worker to eliminate this overhead entirely?

---

## Category 5: Debugging & Production Incidents

**Q16.**
A container is reported as "running" by `docker ps`, but the application inside is not
responding. You are not allowed to restart the container.

List every diagnostic step you would take, in the exact order, specifying the command for
each step and what you are looking for at each step.

---

**Q17.**
You have a distroless container running in production. It starts throwing errors.
`docker exec -it <container> sh` fails with "executable not found."

How do you:
1. Get a shell-like environment in the container's context?
2. Capture a network traffic dump of what the container sends/receives?
3. Read the files inside the container's filesystem?
4. Check what the process is doing at the system call level?

Provide the exact commands for each.

---

**Q18.**
Your monitoring shows a container's memory usage growing by ~50 MB every hour.
After 8 hours it gets OOM killed and restarts.

Describe your complete investigation process:
- How do you confirm the memory growth trend without waiting for the OOM kill?
- How do you identify which part of the application is leaking?
- What does the OOM event look like in `docker events` and `dmesg`?
- What is the immediate mitigation (without fixing the code)?
- What is the permanent fix?

---

## Category 6: Security

**Q19.**
A container is running with:
```bash
docker run -d -v /var/run/docker.sock:/var/run/docker.sock myapp:latest
```

Explain exactly how this creates a container escape vulnerability.
Write the two-line command an attacker inside the container would run to get root access
to the host. What is the correct alternative if the app genuinely needs to interact with Docker?

---

**Q20.**
Compare these two container runs:
```bash
# Run A
docker run --privileged myapp:latest

# Run B
docker run --cap-add SYS_PTRACE --cap-add NET_ADMIN myapp:latest
```

Which is more dangerous and why? What can Run A do that Run B cannot?
If your application only needs to use `strace` for debugging, which capabilities does it
actually need? Name them.

---

**Q21.**
You are doing a security review of a Compose file and find:
```yaml
services:
  api:
    image: myapi:latest
    environment:
      - DB_PASSWORD=prod-db-password-2024
      - JWT_SECRET=supersecretjwtkey
    ports:
      - "0.0.0.0:5000:5000"
```

List every security issue you see. For each issue, explain the specific risk and provide the
corrected configuration or approach.

---

## Synthesis Challenge (Final Boss)

**Q22.**
You are the on-call engineer. You receive this alert at 2 AM:

> "Production API response times spiked from 50ms to 8s. Error rate is 15%.
> One of our 3 API container replicas shows high memory. DB appears healthy.
> No deployments in the last 6 hours."

You have access to the host machine. The API containers are built on `python:3.12-slim`,
run as non-root, have health checks, and are behind an nginx load balancer.

Write your complete incident response plan:
1. What is your first command, and why?
2. How do you identify which of the 3 replicas is the problem instance?
3. Without restarting any container, what would you do to investigate the slow requests?
4. What would make you decide to restart the problematic replica?
5. After the incident, what three improvements would you make to prevent recurrence?
6. Write a 5-line post-incident summary in the format: Impact / Timeline / Root Cause / Fix / Prevention.

---

## Scoring Guide

| Questions answered correctly | Assessment |
|---|---|
| 18–22 | Expert-level container engineer. Ready for senior/staff roles |
| 14–17 | Strong intermediate. Can operate production systems. Minor gaps to fill |
| 10–13 | Solid foundation. Re-do Day 4 and Day 5 exercises |
| 6–9 | Core concepts understood but gaps in networking and debugging. Re-do Day 2 and 5 |
| Below 6 | Go back to Day 1. Foundations need reinforcement before advancing |

---

## Answer Approach Guide

For each question you cannot answer:
1. Identify which Day's content covers it (see SUMMARY.md index)
2. Go back to that specific exercise
3. Run the exercise again with fresh attention to the "why", not just the "what"
4. Come back and answer the question before moving on

The goal is not a score — it is to ensure no concept is a blind spot when you're on-call
at 2 AM with a production container incident and no time to search Stack Overflow.
