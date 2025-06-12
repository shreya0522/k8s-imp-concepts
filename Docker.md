

# =======================================================================================================================================================================================================================================
📚 1. Basics of Docker (Foundational Questions)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
❓ 1. What is Docker, and why do we use it?

Answer: Docker is a containerization platform that packages applications and their dependencies into isolated units called containers.
Why:
- Consistent environments (dev → prod)
- Lightweight, fast, and portable
- Easy scaling and deployment

Follow-up:
👉 How is Docker different from a virtual machine?
- Docker shares the host OS kernel → lightweight
- VMs run their own OS → heavy
- Docker starts in seconds; VMs take minutes

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔍 2. What are the benefits of using Docker over traditional virtualization (VMs)?

Benefits of Docker over traditional VMs
| Feature         | Docker                      | VMs                     |
| --------------- | --------------------------- | ----------------------- |
| Boot time       | Seconds 
                    | Minutes                 |
| Resource usage  | Low (shares host OS kernel) | High (each has full OS) |
| Portability     | High                        | Lower                   |
| Isolation level | Process-level               | Hardware-level          |
| Image size      | Small (\~100MB)             | Large (GBs)             |

Follow-up – Q: Is Docker secure like VMs?
Not by default. VMs offer stronger isolation. But Docker can be secured with tools like AppArmor, SELinux, and rootless containers.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

📦 3. Difference between an Image and a Container

- Image: A blueprint (app + dependencies) – read-only
- Container: A running instance of an image – live and mutable

* Follow-up – Q: Can multiple containers run from one image?
* Yes. Each container is isolated but based on the same image.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧱 4. What is the Docker Engine?

Answer: Docker Engine is the core software that runs and manages containers. It has three parts:
- Docker Daemon (dockerd) – manages containers
- Docker CLI – command-line tool to talk to daemon
- REST API – interface used by Docker tools & GUIs

Follow-up – Q: Is Docker Engine open-source?  Yes, Docker CE (Community Edition) is open-source.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 5. Docker container lifecycle

| Stage         | Command                    | Description                     |
| ------------- | ---------------------------|-------------------------------- |
| Created       | `docker create nginx`      | A container that has been created but not started |
| Running       | `docker start <id>`        | A  container running with all its processes       |
| Paused        | `docker run nginx`         | A container whose processes have been paused      |
| Stopped       | `docker stop <id>`         |  A container whose processes have been stopped             |
| Deleted       | `docker kill <id>`         | A container in a dead state          |


Follow-up – Q: What happens if you remove a running container?
> You’ll get an error. You must stop it before removal (docker rm -f forcefully removes).

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🐳 6. What happens when you run docker run hello-world?
>Workflow:
- CLI sends run command to Docker daemon
- Daemon checks if hello-world image exists
- If not, downloads from Docker Hub
- Creates container from image
- Starts container – prints welcome message
- Container exits (one-shot run)

Follow-up –
 Q: What if Docker Hub is blocked or image doesn't exist?
- If image not found: You get an error.
- If Docker Hub is unreachable: Pull fails unless image is cached locally.

# =======================================================================================================================================================================================================================================
🧱 2. Images and Dockerfile
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

📑 1. What is a Dockerfile and what are its most common instructions?
🧱 Most common instructions:
| Instruction  | Purpose                                                              |
| ------------ | -------------------------------------------------------------------- |
| `FROM`       | Base image (e.g., `FROM ubuntu`)                                     |
| `RUN`        | Execute commands during image build (e.g., `RUN apt update`)         |
| `COPY`       | Copy files from local system to image                                |
| `ADD`        | Like `COPY` but also supports remote URLs & auto-extracting archives |
| `CMD`        | Default command when container runs                                  |
| `ENTRYPOINT` | Sets a fixed command to always run                                   |
| `EXPOSE`     | Documents the port the container will use                            |
| `ENV`        | Set environment variables                                            |
| `WORKDIR`    | Sets working directory for following instructions                    |
| `ARG`        | Set build-time variables                                             |

🧠 Follow-up – Q: Can I use multiple CMDs?
Only the last CMD takes effect. If multiple, the earlier ones are overridden.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔀 2. Difference between CMD vs ENTRYPOINT

- Think of ENTRYPOINT as: “What this container is meant to do, always.” It sets the main command — like saying: “this is a Python script runner” or “this is a web server”.
- Think of CMD as: “What arguments should be passed by default, unless the user says otherwise?” . It sets the default inputs or behavior — like saying: “by default, run this script with argument X, unless someone wants Y.”

✅ Use Case 1: Python Script Runner
Imagine you’re building a container that always runs a Python script named processor.py, but the filename to process can change.
```
FROM python:3.10
COPY processor.py /app/processor.py
WORKDIR /app
ENTRYPOINT ["python", "processor.py"]
CMD ["input1.json"]
```
Behavior:
| Command                           | What happens                            |
| --------------------------------- | --------------------------------------- |
| `docker run my-image`             | Runs: `python processor.py input1.json` |
| `docker run my-image input2.json` | Runs: `python processor.py input2.json` |

✅ ENTRYPOINT defines the command to run, and
✅ CMD gives a default file to process, which you can override.

✅ Use Case 2: Shell Script for Backup
You want a container that always runs a backup.sh script, but sometimes you want to pass different folders to back up.
```
FROM alpine
COPY backup.sh /backup.sh
RUN chmod +x /backup.sh
ENTRYPOINT ["/backup.sh"]
CMD ["/data"]
```
Behavior:
  - docker run backup-tool → Backs up /data
  - docker run backup-tool /mnt/files → Backs up /mnt/files

✅ Good for cron jobs, backup tools, conversion scripts.


❗ Bad Use Case (Where confusion happens)
```
ENTRYPOINT ["echo", "hello"]
CMD ["world"]
```
This looks fine, but:
- docker run image → echo hello world
- docker run image everyone → echo hello everyone
- You can’t override echo easily — you’re stuck with it unless you use --entrypoint.
That’s why use ENTRYPOINT only when your container has one purpose (like: “always run this script”), and use CMD for optional inputs.



Follow-up – Q: Can I use both together?
Yes! ENTRYPOINT runs first, and CMD provides arguments to it.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 3. How does Docker layer caching work during image build?
Answer: Docker builds images in layers. Each instruction in Dockerfile creates a new layer. If a layer hasn't changed, Docker reuses the cache to speed up the build.

📦 Workflow:
- FROM ubuntu → base layer
- RUN apt install curl → next layer
- COPY . /app → new layer
- If you change something at the top, all layers below are rebuilt.

🧠 Tip: Place stable instructions (like RUN apt-get update) early to benefit from caching.

Follow-up – Q: Why is layer order important?
Because Docker rebuilds layers from the first changed line onward.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

❓ 4. Why is COPY preferred over ADD in most cases?
| Aspect        | `COPY`                    | `ADD`                                 |
| ------------- | ------------------------- | ------------------------------------- |
| Purpose       | Only copies files/folders | Also supports URLs, tar files         |
| Simplicity    | Clear intent              | Has extra behavior (can be confusing) |
| Best practice | Preferred for clarity     | Use only if you need extraction or URLs     |

✅ COPY is better because it’s simpler and does only one thing.

Follow-up – Q: When should you use ADD?
Use ADD only if:
  - You need to extract a tar.gz archive
  - You want to download from a URL (though not recommended)

📦 COPY vs ADD in Dockerfile
| Feature                   | `COPY`                                   | `ADD`                                              |
| ------------------------- | ---------------------------------------- | -------------------------------------------------- |
| 🔧 **Primary Use**        | Copies files/directories into container  | Same, but with extra features                      |
| 🗂️ **Source Type**       | Only local files                         | Local files **or** remote URLs/archives            |
| 📦 **Archive Extraction** | ❌ Does **not** extract `.tar` files      | ✅ Automatically extracts `.tar`, `.tar.gz`, `.tgz` |
| 🌐 **Remote URLs**        | ❌ Not supported                          | ✅ Supports URLs (e.g., `ADD http://...`)           |
| 📄 **Syntax**             | `COPY src dest`                          | `ADD src dest`                                     |
| 💡 **Best Practice**      | Preferred for clarity and predictability | Use **only** if you need extraction or URLs        |

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

💡 5. What is the use of .dockerignore?
Answer: .dockerignore is like .gitignore. It tells Docker to ignore files/folders during build context transfer.

🚀 Why it matters:
- Speeds up image build
- Prevents unnecessary files (e.g., .git, node_modules, logs) from being copied into the image
-Reduces final image size

📁 Example contents of .dockerignore:
```
.git
*.log
node_modules
Dockerfile.dev
```
Follow-up – Q: Does .dockerignore affect the running container?
No. It only affects what files are sent to the Docker daemon during build.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧼 6. How to write an optimized Dockerfile for smaller image size?
✅ Best practices:
- Use slim or alpine base images ```FROM node:18-alpine ```
- Combine commands with && to reduce layers:
``` RUN apt update && apt install -y curl && rm -rf /var/lib/apt/lists/* ```
- Use .dockerignore to exclude unnecessary files
- Avoid installing dev tools in prod images
- Clean up cache/temp files in the same RUN step
- Minimize use of ADD
- Prefer multi-stage builds for production

🔄 Multi-stage example:
```
FROM golang:1.20 as builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM alpine
COPY --from=builder /app/myapp /myapp
ENTRYPOINT ["/myapp"]
This produces a minimal final image with only the binary.
```

Follow-up – Q: Why does using Alpine reduce size?
Because Alpine is a minimal Linux distro (~5MB), compared to Ubuntu (~70MB).

# =======================================================================================================================================================================================================================================
🛠️ 3. Container Lifecycle & Management
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


🎮 1. How do you create, start, stop, and delete a container?
📌 Commands & Workflow:
| Action      | Command                            | Notes                                         |
| ----------- | ---------------------------------- | --------------------------------------------- |
| Create only | `docker create <image>`            | Creates container but doesn't start it        |
| Start       | `docker start <container-id/name>` | Starts a **created** or **stopped** container |
| Stop        | `docker stop <container-id/name>`  | Gracefully stops running container            |
| Delete      | `docker rm <container-id/name>`    | Removes a **stopped** container               |


✅ Example:
```
docker create --name myapp nginx         # Creates container
docker start myapp                       # Starts it
docker stop myapp                        # Stops it
docker rm myapp                          # Deletes it
```
🔁 Follow-up Q:
Q: Can you delete a running container?
A: No, you must stop it first, or use docker rm -f <container> to force remove.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔄 2. How do you restart a container that exited unexpectedly?
Option 1: Manual ```docker start <container-name>```
Option 2: Automatically using --restart flag: ``` docker run --restart=always <image> ```
🧠 Restart Policies:
- no (default): never restart
- on-failure: restart only on error exit
- always: restart always, even if stopped manually
- unless-stopped: restart unless user stopped

🔁 Follow-up Q:
Q: How do you check why a container exited?
```
docker ps -a      # Shows exited containers
docker logs <container-name>
```
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧠 3. What happens when you run docker run without -d or -it?
Answer:
- No -d (detached): container runs in the foreground
- No -it (interactive terminal): you can’t interact with terminal input/output (like bash)

Example:
```
docker run ubuntu                     # Runs & exits (no interactive terminal)
docker run -it ubuntu bash           # Opens shell in interactive mode
docker run -d nginx                  # Runs nginx in background
```
🧠 If no terminal (-it) is attached and the container needs it (like bash), it exits quickly.

🔁 Follow-up Q:
Q: How do you detach from an interactive container safely?
Press Ctrl + P then Ctrl + Q.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧹 4. What does the --rm flag do when running a container?
Answer: It automatically deletes the container after it stops. 📦 Useful for temporary containers or one-time scripts.
``` docker run --rm ubuntu echo "hello" ```
✅ Saves you from cleaning up stopped containers manually.

🧠 Tip:
Don't use --rm with containers you want to debug or inspect after they exit.

🔁 Follow-up Q:
Q: Can you use --rm with -d (detached)?
Yes, but it's risky—container might get removed before you check logs.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

⚠️ 5. Why might a container exit immediately after starting?
Common Reasons:
- No long-running process – image runs a command and exits (e.g., echo, ls)
- Error in entrypoint or command
- App crashed due to missing dependencies
- No interactive terminal (-it needed)

Example:
```
docker run ubuntu             # Exits immediately (no command to keep it alive)
docker run ubuntu sleep 30    # Stays running for 30 seconds
```

🛠️ To debug:
```
docker logs <container>
docker inspect <container>
```

🔁 Follow-up Q:
Q: How do you keep a container running?
Use a long-running command like sleep infinity, Use proper entrypoint (like a web server)

# =======================================================================================================================================================================================================================================
🌐 4. Docker Networking
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🔌 1. What is the default bridge network, and how does it work?
Answer: The default bridge network is created by Docker (bridge) when it installs. When you run a container without specifying a network, it connects to this default bridge network. It uses NAT (Network Address Translation) to allow containers to communicate with the host and the internet.
Each container gets a private IP (e.g., 172.17.x.x).

Workflow:
```
docker run nginx          # connects to default bridge
```
You can ping containers by IP, but not by name in default bridge.

For name resolution, use custom bridge network.

Follow-up Q:
🟡 How do containers resolve names in a custom bridge?
👉 Docker adds internal DNS when using user-defined bridge networks, so containers can talk via names

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🌍 2. Difference between bridge, host, overlay, and none network drivers?
| Driver    | Use Case                         | Behavior                                             |
| --------- | -------------------------------- | ---------------------------------------------------- |
| `bridge`  | Default for single-host setup    | Private network on host, NAT access to world         |
| `host`    | High-performance networking      | Container shares host’s network stack (no isolation) |
| `overlay` | Multi-host communication (Swarm) | Creates virtual network across nodes                 |
| `none`    | Completely isolated              | No network at all, manual control needed             |

Follow-up Q:
🟡 When should I use host networking?
👉 Use it for performance-sensitive apps (e.g., load balancer), but loses isolation.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔎 3. How does DNS resolution happen between Docker containers?
Answer: Docker has an internal DNS server when using user-defined bridge or overlay networks. Containers can talk using container names.

Example:
```
docker network create mynet
docker run -dit --name app1 --network mynet nginx
docker run -dit --name app2 --network mynet busybox
```
Now inside app2, you can:
```
ping app1     # resolves via Docker's internal DNS
```
Follow-up Q:
🟡 Why doesn’t name resolution work in default bridge?
👉 Because default bridge doesn’t register names in internal DNS.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🛠️ 4. How do you connect containers across different networks?
You can connect a container to multiple networks using:
``` docker network connect <network> <container> ```
Example:
```
docker network create net1
docker network create net2
docker run -dit --name c1 --network net1 nginx
docker network connect net2 c1     # Now c1 is on both net1 and net2
```
Now c1 can communicate with containers in both networks.

Follow-up Q:
🟡 Can a container be in more than one network?
👉 Yes, Docker allows multi-homed containers.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

📡 5. What happens if two containers try to expose the same port?
Answer: If both containers try to bind the same host port, Docker throws an error.

Example:
```
docker run -d -p 8080:80 nginx      # OK
docker run -d -p 8080:80 httpd      # Fails: port already in use
Note: Exposing same container port (e.g., 80) is fine as long as host ports differ.
```

Workaround:
```
docker run -d -p 8080:80 nginx
docker run -d -p 8081:80 httpd     # OK
```
Follow-up Q:
🟡 Can you bind to different IP addresses for same port?
👉 Yes, on different IPs on host (advanced setup).


# =======================================================================================================================================================================================================================================
🧪 5. Docker Compose
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Q - "Why can't we just use traditional Docker CLI (docker run, docker network, etc.) instead of Docker Compose?"
 A - ❓Why Not Just Use Docker CLI?
You can, but using plain docker run becomes unmanageable and error-prone as soon as your app involves:
 - Multiple containers
 - Custom networks, volumes, environment variables
 - Port mappings, healthchecks, restart policies
 - Different setups for dev, test, and prod

⚠️ Problem with Traditional Docker CLI:
| Problem                          | Why It’s Painful                                                            |
| -------------------------------- | --------------------------------------------------------------------------- |
| 🧱 **No single source of truth** | You have to document setup in README or scripts.                            |
| 🔄 **Manual coordination**       | You manually start each container (`docker run ...`), in the correct order. |
| ❌ **Hard to share or reproduce** | Others need to repeat all your CLI steps on their machine.                  |
| ⚙️ **No easy restart/cleanup**   | You must track and stop containers manually.                                |
| 🌎 **Not environment portable**  | Changing ports, paths, or volumes requires changing many CLI flags.         |

✅ Why Docker Compose is Better (as per official doc)
1. 🔧 Simplified control (Single YAML definition)
 - You define all containers, configs, ports, volumes, and networks in one docker-compose.yml file.
 - One command (docker compose up) starts everything.
🧱 With CLI, you'd need multiple docker run, docker network create, and docker volume create commands — all manually coordinated.

2. 🤝 Efficient collaboration
- The Compose file is easy to share, just like code.
- No need to write long “Getting Started” setup instructions.
- Everyone uses the same config on dev/stage/CI.
🧱 With Docker CLI, each developer or team would need to run multiple commands, or write scripts, which may get out of sync.

3. ⚡ Rapid development (Container re-use)
- Compose caches containers. If a service hasn’t changed, Compose re-uses it.
- Speeds up restarting services (e.g., just restart web without re-creating db).
🧱 With plain Docker CLI, you’d likely stop/remove containers, then re-run them entirely — slower and clunkier.

4. 🌍 Portability via environment variables
- Compose supports .env files and ${VARIABLE} references in docker-compose.yml.
- Same file can be reused for:
           * Dev (localhost ports, dev volumes)
           * CI (different ports, mock services)
           * Prod (different images, secrets, storage)
🧱 With Docker CLI, you'd need custom shell scripts or export dozens of env vars.

5. 💥 Single-host deployments
- Docker Compose can handle simple production deployments on one host.
- Useful for internal tools, dashboards, monitoring stacks, or sidecar apps.
🧱 Docker CLI can also do this — but Compose does it more cleanly and repeatably.

6. 🧪 Automated testing environments
- Compose can spin up full app stacks in CI:
  ```
  docker compose up -d
  ./run_tests
  docker compose down
  ```
Clean environment created, tested, and destroyed in 3 lines.
🧱 With Docker CLI, test orchestration scripts get bloated and fragile.

🔄 Real Example (Node.js + Redis)
✅ With Compose:
```
version: "3.8"
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - redis

  redis:
    image: redis:alpine
```
``` docker compose up ```

❌ Same with CLI:
```
docker network create app-net
docker run -d --name redis --network app-net redis:alpine
docker build -t myapp .
docker run -d --name web --network app-net -p 3000:3000 myapp
```
It works — but harder to manage, debug, and share.

✅ Final Summary
| Aspect                        | Docker CLI           | Docker Compose               |
| ----------------------------- | -------------------- | ---------------------------- |
| Config stored as code         | ❌ Manual             | ✅ YAML file                  |
| Multi-container orchestration | ❌ Tedious            | ✅ Native                     |
| Reusability                   | ❌ No                 | ✅ High                       |
| Environment handling          | ❌ Limited            | ✅ `.env`, variable overrides |
| Portability & CI/CD           | ❌ Requires scripting | ✅ Built-in support           |
| Cleanup / teardown            | ❌ Manual             | ✅ `docker compose down`      |

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧱 1. What is Docker Compose and why is it used?
Answer:Docker Compose is a tool that helps you define and run multi-container applications using a YAML file (docker-compose.yml).
Why use it?
- Manage multiple services (e.g., web + DB) easily.
- Define everything (networks, volumes, dependencies) in one file.
- One command to start/stop everything.

Follow-up Q:
🟡 How is it different from plain Docker CLI?
👉 Compose manages multi-container dependencies, networks, and lifecycle together.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🧬 2. What does a typical docker-compose.yml file look like?
```
version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: example
```
Key Concepts:
- services: define containers.
- networks and volumes can also be defined.
- All services are in the same network by default (can use container names).

Follow-up Q:
🟡 Can we specify volume mounts and environment files?
👉 Yes, you can use volumes: and env_file: keys.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🔁 3. Difference between docker-compose up, up --build, and up -d
| Command                     | Meaning                                        |
| --------------------------- | ---------------------------------------------- |
| `docker-compose up`         | Starts services (builds if needed, shows logs) |
| `docker-compose up --build` | **Force rebuilds** images before starting      |
| `docker-compose up -d`      | Runs in **detached** mode (runs in background) |
Tip: Use --build when your Dockerfile or context has changed.

Follow-up Q:
🟡 How to stop services?
👉 Use docker-compose down.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

⚠️ 4. What happens if one service fails during docker-compose up?
Answer: If a critical service (like a DB) fails:
- Other services may fail if they depend on it (e.g., can't connect).
- The container is marked exited but Compose keeps others running.

Use depends_on: to define order, but note: It doesn’t wait for the service to be "ready", just until it's started.

Follow-up Q:
🟡 How to make services wait for DB?
👉 Use wait-for-it.sh or similar script to check readiness.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🧩 5. How does Compose manage environment variables across services?
Answer: You can define env vars in 3 ways:

>> Inside docker-compose.yml:
```
environment:
  - DB_USER=root
  - DB_PASS=secret
```

>> Using .env file in the same directory:
```
DB_USER=root
DB_PASS=secret
```

>> Using env_file in Compose:
```
env_file: .env
```

Variable Reference: You can also reference them using ${VAR_NAME}:
```  image: ${APP_IMAGE} ```

Follow-up Q:
🟡 Which method takes priority?
👉 Vars defined directly in docker-compose.yml override .env file.


# =======================================================================================================================================================================================================================================
💾 6. Storage & Volumes
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
📦 1. What is the difference between volumes and bind mounts?
Both are ways to persist data or share files between your host system and a Docker container.
But they work differently under the hood and are used for different purposes.

🧱 A. Volumes (Docker-managed)
- Docker creates and manages the storage.
- The data is stored under ``` /var/lib/docker/volumes/<volume-name>/_data ```
- You don’t care where it lives — Docker handles it.
✅ Best for:
- Databases (/var/lib/mysql) , - Upload directories , - Persistent container data
🟢 Example:
```
docker volume create mydata
docker run -v mydata:/app/data myimage
```

📂 B. Bind Mounts (Host-managed)
- You tell Docker exactly which folder on the host to mount.
- Docker uses the exact path you provide.
🟡 Risk: The container now has access to host files directly.
✅ Best for:
- Local dev (e.g., live-mounting source code)
- Mounting config files or logs
- Fast, manual testing
🟢 Example: ``` docker run -v /home/yash/code:/app myimage ```
- Now the container's /app folder is the real host folder /home/yash/code.
- Changes inside the container = changes on your real machine, and vice versa.

what is the meaning that " 🟡 Risk: The container now has access to host files directly." 
When you use a bind mount, you are telling Docker: “Attach this specific folder from my host machine into the container.” For example: ``` docker run -v /home/yash/code:/app my-image```
This means:
- Whatever files are in /home/yash/code on your real machine
- Will appear inside the container at /app
- Any changes made in the container to /app will change your real files on the host

So the container has direct access to your host’s files and can:
- 📝 Modify them
- 🗑️ Delete them
- 📤 Read sensitive data from them

⚠️ Why is that risky?
When you use volumes, Docker manages where data is stored. The container has access only to that volume, and it's isolated in /var/lib/docker. When you use bind mounts, the container touches your real filesystem.
This can be dangerous if:
- The container has bugs or malicious code
- It has write access to config files, secrets, or system directories
- You accidentally bind-mount a sensitive directory like /etc or /var
exp : ``` docker run -v /:/host alpine```
Inside that container: ``` rm -rf /host/etc```  Boom 💣 — you just deleted your host's /etc.

| Feature         | Volume                 | Bind Mount                  |
| --------------- | ---------------------- | --------------------------- |
| Data path       | Docker-managed         | Host-specified              |
| Security        | Isolated               | Full access to host         |
| Visibility      | Only visible to Docker | Visible to container + host |
| Safe by default | ✅ Yes                  | ❌ Risky if misused          |

📌 Volumes are the Docker-native way.
📌 Bind mounts give full access to host paths (more flexible, but riskier).

Follow-up Q:
🟡 Can I use both in one container? → Yes, you can mount both at once.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
🔐 2. When would you use named volumes vs anonymous ones?
Answer - 
🔍 What are Docker volumes?
Volumes are storage locations outside the container's writable layer. They let you: - Persist data (e.g., databases) , - Share data between containers , - Keep data even if the container is deleted

There are two types of volumes you can create with -v:
🅰️ Named Volumes: 
```
docker volume create mydata
docker run -v mydata:/app/data myapp
```
OR:
```
docker run -v mydata:/app/data myapp
```
(If mydata doesn’t exist, Docker creates it automatically.)

🔑 What happens:
- Docker creates a volume named mydata
- It's stored under /var/lib/docker/volumes/mydata/_data
- You can reference it by name across multiple containers

✅ Why it’s useful:
- Easy to identify, back up, and inspect
- Can be reused across runs or containers
- Works well in CI/CD, backups, or database storage

✅ Use When:
- You want to persist data across container restarts
- You want to share storage between containers
- You want to manage it yourself (inspect, delete, copy)

🅱️ Anonymous Volumes
``` docker run -v /app/data myapp ``` 
You did not provide a name before the colon.

🔑 What happens:
- Docker creates a random name like 1e2f3g...
- Also stored in /var/lib/docker/volumes, but no easy way to refer to it later
- It’s used for that run of the container

⚠️ Why it’s risky or harder:
-  You don’t know the name, so:
      * You can't easily clean it up
      *  You can't reuse it
      *  You can forget it exists
- It creates clutter and orphan volumes (leftovers taking disk space)

❌ Use only when:
- You don’t care about the data after container stops
- You want quick scratch space
- You’re prototyping or testing something short-lived

✅ Summary Table
| Feature       | Named Volume                                       | Anonymous Volume                           |
| ------------- | -------------------------------------------------- | ------------------------------------------ |
| Creation      | `docker volume create mydata` or `-v mydata:/path` | `-v /path` (no name given)                 |
| Reusability   | ✅ Can reuse across containers                      | ❌ Can't reuse easily                       |
| Manageability | ✅ Easy to list, inspect, back up                   | ❌ Hard to find or manage                   |
| Default Name  | You provide (`mydata`)                             | Docker generates (`a1b2c3...`)             |
| Cleanup       | Easy (`docker volume rm mydata`)                   | Hard — must inspect container to find name |
| Best Use Case | Databases, persistent uploads, shared cache        | Temporary data, throwaway test runs        |

💡 Rule of Thumb (Easy to remember)
✅ Use named volumes when data needs to live after the container dies. ❌ Avoid anonymous volumes unless you’re doing short-lived throwaway tasks.

🔍 How to check what's created
List all volumes: ```docker volume ls```
Inspect one: ```docker volume inspect mydata```
Clean up anonymous volumes: ```docker volume prune```

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔁 3. Can volumes be shared between multiple containers?
✅ Yes. You can mount the same volume into multiple containers.

Example:
```
docker run -v shared:/data app1
docker run -v shared:/data app2
```
Use case:
Two containers reading/writing to same data (like nginx + app logs, or app + sidecar).

Follow-up Q:
🟡 Any risk?
👉 Yes, race conditions or file locks if both containers write simultaneously.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔥 4. What happens to data in a volume if the container is removed?
If you remove the container, volume data is not deleted. But if you use --rm or docker-compose down -v, the volume can be deleted.
🔹 Volumes outlive containers unless explicitly removed.

Follow-up Q:
🟡 How to clean unused volumes?
👉 Run: docker volume prune

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 

🗃️ 5. Where are Docker volumes stored on the host?
📍 Default location on Linux:
```
/var/lib/docker/volumes/
```
Each volume has a subdirectory with its own UUID.

Follow-up Q:
🟡 Can I inspect volume contents from the host?
👉 Yes, but use caution:
```
sudo ls /var/lib/docker/volumes/<volume_name>/_data
```
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- 

❓Will Docker create a volume by default when I create a container?
✅ Yes — but only if the image declares a VOLUME instruction in its Dockerfile
or you use -v / --mount in your docker run command.

🔧 1. If the Docker image has a VOLUME instruction
For example, the Dockerfile for MySQL has: ```VOLUME /var/lib/mysql``` Then when you run a container from this image: ```docker run -d --name mydb mysql```
Docker will automatically: - Create an anonymous volume , - Mount it to /var/lib/mysql in the container , - The volume’s name will be auto-generated (e.g., f2e4dfc2d7...)
✅ You didn’t provide -v, but Docker still created a volume because the image told it to.

🔧 2. If you manually pass -v or --mount:
```docker run -v mydata:/data myapp``` 
✅ You control what kind of volume is created: - If mydata exists → reuse it , - If not → Docker creates it as a named volume

❌ 3. If the image doesn’t declare a volume and you don’t pass -v 
Then no volume is created at all. The container uses: - The writable layer (temporary) - Data inside the container is lost when the container is deleted
Example: ```docker run -d --name c1 alpine``` 
- No volumes created
- Any files written inside the container will be lost after docker rm c1

📌 Summary
| Situation                                 | Will volume be created? | Type                |
| ----------------------------------------- | ----------------------- | ------------------- |
| Image has `VOLUME` in Dockerfile          | ✅ Yes                   | Anonymous volume    |
| You use `-v` or `--mount` in `docker run` | ✅ Yes                   | Named or bind mount |
| No `VOLUME` in image and no `-v`          | ❌ No                    | No volume at all    |

# =======================================================================================================================================================================================================================================
🧠 8. Advanced Concepts
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔐 1. What is Docker Content Trust (DCT)?
Answer: Docker Content Trust (DCT) is a security feature that ensures you're only using verified, signed container images — not something fake, tampered, or injected with malicious code.

🔧 How it works:
- When DCT is enabled (DOCKER_CONTENT_TRUST=1), Docker only pulls signed images.
- It uses Notary under the hood to sign and verify images.
- The image publisher signs the image.
- The consumer (user) verifies the signature before using it.

Here’s what happens when DCT is enabled:
- The image publisher signs the image using cryptographic keys (via Docker’s Notary service).
- You, the user, try to pull the image: ``` export DOCKER_CONTENT_TRUST=1``` ```docker pull nginx```
- Docker checks:
     * “Is this image digitally signed by the publisher?”
     * “Is the signature valid and unchanged?”
- If the image is signed and valid → ✅ Pull succeeds
- If the image is not signed → ❌ Pull fails with an error

🧪 Real Example
```
export DOCKER_CONTENT_TRUST=1
docker pull alpine
```
If the alpine image has a valid signature, Docker pulls it.
If you try:
``` docker pull my-unsigned-image ```
And it’s not signed, Docker will say: "Error: image is not signed or signature is invalid."

📌 Why is DCT important?
| Problem                                | DCT helps solve                                    |
| -------------------------------------- | -------------------------------------------------- |
| 🕵️‍♂️ Pulling fake or tampered images     | Stops unsigned or altered images from being used   |
| 🔐 Supply chain attacks                | Verifies the source of images                      |
| 🧯 Accidental image spoofing           | Prevents you from pulling wrong/untrusted versions |

📌 Why it matters:
- Prevents pulling tampered or unauthorized images.
- Adds a layer of supply chain security.

❓ Follow-up Question:
What happens if you try to pull an unsigned image with DCT enabled?
👉 The pull fails with an error saying the image is not signed.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🔒 2. How do you run a container as a non-root user?
By default, Docker containers run processes as root inside the container, even if your host user isn’t root. Running as root inside the container can be risky — especially if an attacker breaks out of the container.
✅ The secure solution: Run your container’s main process as a non-root user.

🔧 A Option A: Use the --user flag in docker run
Example: ``` docker run --user 1001:1001 myimage```
    * 1001 is the UID (user ID) , * 1001 is the GID (group ID) , 
This tells Docker to run the container not as root, but as user ID 1001

You can also use usernames if the image defines them: ``` docker run --user node node:alpine ``` 

🔧 B Option B: Create a non-root user in the Dockerfile
Add this to your Dockerfile:
```
FROM node:18-alpine

# Create non-root user and group
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Use the new user
USER appuser

WORKDIR /app
COPY . .

CMD ["node", "app.js"]
```
✅ Now when the container runs, it defaults to appuser, not root.

🔧 C- Option C: Use an official image that already runs as non-root
Some images (like node, nginx, postgres) come with built-in non-root users (e.g., node, nginx, postgres). Check their docs and do: ```docker run --user postgres postgres```

🔍 How to check which user is running inside the container
```docker exec -it <container_name> whoami``` or ```docker exec -it <container_name> id```
If you see root — you’re running as root.If you see something like appuser (uid=1001) — you're good.

⚠️ Why this matters (security)
| Risk                  | If running as root                                |
| --------------------- | ------------------------------------------------- |
| Breakout attacks      | Can affect host system                            |
| File permission abuse | Can read/write unintended files (if bind-mounted) |
| Compliance issues     | Violates least privilege principle                |

✅ Summary
| Method                            | How                                   |
| --------------------------------- | ------------------------------------- |
| `--user` flag                     | `docker run --user 1001:1001 myimage` |
| Dockerfile `USER`                 | Add `USER appuser` after creating it  |
| Use base image with built-in user | E.g., `node`, `nginx`, `postgres`     |

❓ Follow-up Question:
What issues might occur with non-root users?
👉 Permissions problems when accessing mounted volumes or system files.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

🧪 3. What are multi-stage builds and why are they important?
Answer:Multi-stage build is a Docker feature that lets you:
- Use multiple FROM statements in the same Dockerfile
- Separate build logic from runtime
- Copy only the necessary artifacts from one stage to another
✅ This results in a smaller, cleaner, and more secure final image.

🧠 Why it’s useful:
Normally, if you build an app (like compiling TypeScript, Go, Java), your image ends up bloated with: - Build tools , - Compilers , - Source code
With multi-stage builds: You compile in one image, then copy only the built output into a slim final image. 

🧱 Scenario: Maven-based Java Web App
Suppose you have a Spring Boot or plain Java Maven app with this layout:
```
myapp/
├── pom.xml
└── src/
    └── main/java/...
```
❌ Dockerfile WITHOUT Multi-Stage Build
```
# Single stage build (bad for production)
FROM maven:3.8-openjdk-17

WORKDIR /app
COPY pom.xml .
COPY src ./src

# Build the app
RUN mvn clean package

# Run the app
CMD ["java", "-jar", "target/myapp.jar"]
```
🔍 What happens:
Final image contains: - Entire source code , - Maven, JDK, local .m2 repo, - Build tools and your app
📦 Image size: Easily over 1.5–2 GB
📉 Problems:
  - Huge and slow to pull/push, - Insecure — has tools that could be exploited, - Contains sensitive source code

✅ Dockerfile WITH Multi-Stage Build (Best Practice)
```
# --- Stage 1: Build using Maven ---
FROM maven:3.8-openjdk-17 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests

# --- Stage 2: Lightweight runtime ---
FROM openjdk:17-jdk-slim

WORKDIR /app

# Copy only the final JAR from the builder stage
COPY --from=builder /app/target/myapp.jar ./myapp.jar

# Run the app
CMD ["java", "-jar", "myapp.jar"]
```
🔍 What happens:
Stage 1: Compiles your code using Maven and JDK

Stage 2: Uses a minimal image (just enough to run Java)

Only the JAR file is copied to final image

📦 Image size: As small as 250–300 MB


🔎 Side-by-Side Comparison
| Feature                | Without Multi-Stage | With Multi-Stage                    |
| ---------------------- | ------------------- | ----------------------------------- |
| Base Image             | `maven:3.8` (heavy) | Final stage: `openjdk-slim` (light) |
| Build Tools Inside     | ✅ Yes               | ❌ No (only runtime)                 |
| Source Code Inside     | ✅ Yes               | ❌ No                                |
| Image Size             | ❌ 1.5–2 GB          | ✅ 250–300 MB                        |
| Best for Production    | ❌ No                | ✅ Yes                               |
| Security & Portability | ❌ Low               | ✅ High                              |

✅ TL;DR
- Without multi-stage = all-in-one, bloated image that includes build tools and source
- With multi-stage = clean, small, production-ready image that contains only the runnable .jar

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

⚙️ 4. What are namespaces and cgroups, and how does Docker use them?
Namespaces (Isolation): They isolate resources between containers.
| Namespace | Isolates                 |
| --------- | ------------------------ |
| PID       | Process IDs              |
| NET       | Network interfaces       |
| UTS       | Hostname                 |
| MNT       | Mount points/filesystems |
| IPC       | Interprocess comms       |
| USER      | User and group IDs       |

📦 Docker uses namespaces to make containers feel like separate machines.
- cgroups (Control Groups):
They limit and monitor resource usage (CPU, memory, I/O).
Example: --memory=512m --cpus=1.5 limits container resource usage.

📌 Key Use:
8 Prevent containers from starving the host.
8 Enforce QoS and limits.

❓ Follow-up:
What happens if a container exceeds memory limit?
👉 The container gets killed (OOMKilled).

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

.

🧱 5. What’s the role of containerd and how is it related to Docker?
Answer:containerd is a core container runtime that Docker uses under the hood to manage containers.

🧰 Responsibilities of containerd:
 * Pulling images from registries
 * Creating and starting containers
 * Managing storage and networking
 * Exposing an API for higher-level tools

🔗 Docker → containerd → runc:
```
Docker CLI
   ↓
Docker Engine
   ↓
containerd
   ↓
runc (low-level runtime to run containers)
```

📌 Why Docker uses containerd:
* Separation of concerns.
* containerd is CNCF-graduated and can be used by other platforms (like Kubernetes).

❓ Follow-up:
Can containerd run containers without Docker?
👉 Yes. Tools like nerdctl or Kubernetes CRI plugins use containerd directly.

# =======================================================================================================================================================================================================================================
☁️ 9. Docker in Production / CI-CD
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
1- 🏗️ How would you build and push a Docker image in a CI pipeline (e.g., GitHub Actions)?
✅ Prerequisites:
 - Jenkins must be running on a node with Docker installed.
 - Docker daemon must be accessible by the Jenkins user.
 - Docker credentials (username/password or token) should be stored as Jenkins credentials (e.g., dockerhub-creds).
 - Jenkins must have the Docker Pipeline plugin installed.

✅ Jenkinsfile (Declarative pipeline)
```
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'yourdockerhubusername/yourimage'
        IMAGE_TAG = 'latest'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        echo 'Logged in to Docker Hub'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
                        def customImage = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        customImage.push()
                    }
                }
            }
        }
    }
}
```

🔄 Follow-up Interview Questions and Answers
🔸 Q: How do you securely pass Docker credentials to Jenkins?
A: Use Jenkins Credential Manager:
Go to Manage Jenkins > Credentials.
Add Docker Hub username and password/token as "Username with password".
Reference that ID in the pipeline (dockerhub-creds in this case).

🔸 Q: How would you tag images with Git commit SHA or build number in Jenkins?
A:
```
environment {
    IMAGE_TAG = "${env.BUILD_NUMBER}"  // or "${env.GIT_COMMIT.take(7)}"
}
```
This helps you track versions precisely.

🔸 Q: How do you reduce the size of the Docker image during Jenkins builds?
A:
- Use multi-stage builds.
- Use minimal base images like alpine.
- Clean up apt-get, temp files inside the Dockerfile.
- Use .dockerignore to exclude unnecessary files.

🔸 Q: How do you automatically clean old Docker images in Jenkins?
A: Set up a periodic cleanup job using a cron-based Jenkins job:
```docker image prune -af ```
Or use docker system prune with caution.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

2- 🔄 How do you update containers with zero downtime using Docker?
You can achieve zero-downtime deployments using the following methods:
Option 1: Docker Swarm or Kubernetes: Use rolling updates (e.g., docker service update in Swarm or RollingUpdate strategy in K8s).
Option 2: Blue-Green Deployment - Run a new version alongside the old one. , -Switch traffic via load balancer (like Nginx or ALB) once healthy.
Option 3: Canary Deployment - Gradually direct a portion of traffic to new containers.

Follow-up Questions:
🔹 Q: How do you check container health before routing traffic?
A: Use HEALTHCHECK in Dockerfile or K8s readinessProbes.

🔹 Q: How to rollback if something goes wrong?
A: Keep previous image version and redeploy with docker run or use docker service rollback.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

3 - 📉 What are some best practices for Docker in production environments?
- Use minimal base images (like alpine, distroless) to reduce attack surface.
- Pin versions of images (python:3.10.2 instead of latest).
- Avoid running as root.
- Use .dockerignore to exclude unnecessary files.
- Use multi-stage builds to separate build and runtime.
- Implement logging and monitoring via sidecars or logging drivers.
- Tag images properly (e.g., v1.0.1, build-213).
- Use a private registry for sensitive apps.
- Enable security scanning (like Docker Hub scan or Trivy).
- Rotate secrets and avoid hardcoding them in images.

Follow-up Questions:

🔹 Q: What’s the purpose of .dockerignore?
A: Prevents unnecessary files from being copied into the image, speeding up build and reducing size.

🔹 Q: Why avoid using latest tag?
A: It's not predictable — may pull a new version and break deployments.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

4- 🔁 How do you roll back to a previous image version?
Method 1: Manual rollback ``` docker run yourimage:previous-tag ```
Method 2: CI/CD rollback pipeline:  Trigger deployment with a previous version tag.
Method 3: GitOps tools: Use ArgoCD or Flux to revert Git changes and redeploy.

Follow-up Questions:
🔹 Q: How do you ensure old image versions are available?
A: Keep them in the registry with proper tagging policy and avoid auto-pruning.

🔹 Q: How can automation help in rollback?
A: CI tools can maintain a changelog of deployments; if a health check fails, auto-rollback using previous commit.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

5 - 🛑 What are the risks of running containers as root in production?
a- Privilege Escalation — if the container is compromised, the attacker may gain root access to the host.
b- Insecure file access — could overwrite host files if volumes are mounted.
c- Breaks the principle of least privilege — increased attack surface.
d- Violates security compliance (like PCI-DSS, HIPAA).

Alternatives:
- Use USER directive in Dockerfile.
- Use runtime restrictions like Seccomp, AppArmor, SELinux.

Follow-up Questions:
🔹 Q: How to run a container as non-root?
A:
```
FROM node:18-alpine
RUN addgroup app && adduser -S -G app app
USER app
```
🔹 Q: How do Kubernetes and Docker differ in handling security for containers?
A: Kubernetes uses PodSecurityPolicies or OPA/Gatekeeper to enforce restrictions like non-root users; Docker uses runtime profiles and user namespaces.


# =======================================================================================================================================================================================================================================
🔍 10. Real-World Scenarios (Twisted / Practical)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

1- ⚠️ A container keeps restarting — how do you debug it?
Step-by-step debugging:
a -Check restart policy: ``` docker inspect <container_id> | grep -i restart```
b- Check logs: ``` docker logs <container_id>```
c- Check Dockerfile for errors: ``` CMD/ENTRYPOINT may fail immediately (e.g., script not found).```
d- Look at healthcheck failure:  If healthcheck fails continuously, Docker may restart the container.
e- Try running interactively: ```docker run -it --entrypoint /bin/bash your_image```
Follow-up Q:
🔹 Q: What's the difference between RestartPolicy and Healthcheck?
A: RestartPolicy tells Docker when to restart the container. Healthcheck marks it as unhealthy but doesn’t restart unless orchestrated externally (e.g., in Swarm or K8s).

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
2- ❌ You built a new image, but changes are not reflecting — why?
Possible reasons:
- Old image is still being used (you didn’t tag/push/pull properly).
- Container is using a cached layer — use --no-cache to force rebuild:
```
docker build --no-cache -t myimage:latest .
```
- Wrong image tag is being used in deployment.
- Volume is masking changes — e.g., host-mounted volume overwriting new files from image.

Follow-up Q:
🔹 Q: How can Dockerfile layer caching cause this?
A: If your COPY or RUN steps haven’t changed, Docker reuses previous layers. So your code changes may not be reflected.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

3- 🌐 Two containers on the same host can’t communicate — what might be wrong?
Possible issues:
- Different Docker networks:
     *  Use docker network inspect <network> to check.  *  Default bridge does not allow DNS-based name resolution.
- Wrong hostname or container name used.
- No exposed or published port (especially with --network=host vs bridge).
- Firewall rules or SELinux/AppArmor.

Follow-up Q:
🔹 Q: What’s the difference between bridge and host network modes?
A: bridge: Container gets its own IP; DNS-based communication possible.
host: Shares host’s network namespace; better performance but fewer isolation benefits.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
4- 📦 Why is your image suddenly 1.5 GB larger after a small code change?
Common causes:
- Apt cache not cleaned in Dockerfile: ``` RUN apt-get update && apt-get install -y xyz && rm -rf /var/lib/apt/lists/* ```
- Large files accidentally copied (check .dockerignore).
- New dependencies added (e.g., Python packages or node modules).
- Old layers remain in history if image is built without --squash.

Build context too large — run: ```du -sh . ```

Follow-up Q:
🔹 Q: What’s .dockerignore for?
A: Prevents unnecessary files (e.g., .git, node_modules, test files) from being sent to Docker daemon during build, which reduces image size and build time.

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

5- 🐳 Docker says “no space left on device” — what now?
Steps to fix:
- Prune unused resources: ```docker system prune -a```
- Clear dangling images, containers, volumes: 
``` 
docker image prune
docker container prune
docker volume prune
```
- Check disk usage: ```docker system df```
- Move Docker storage location (if space on another disk):
- Update --data-root in Docker daemon config.
- Rotate logs:
If /var/lib/docker/containers/*-json.log is large, add log rotation in /etc/docker/daemon.json:
```
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
Follow-up Q:
🔹 Q: What is overlay2 and how is it related to disk space issues?
A: overlay2 is Docker’s default storage driver; layers can accumulate over time if not cleaned, consuming disk even when containers/images are deleted improperly.


-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------






















