# Dockerfile Instruction: `ARG` (Deep Dive)

`ARG` stands for **Build Argument**.

It allows you to **pass values to a Docker build at build time**. These values are available **only while the image is being built**, not when the container runs (unless you explicitly copy them into the image).

---

# Why do we need `ARG`?

Without `ARG`, values are hardcoded.

Example:

```dockerfile
FROM ubuntu:24.04

RUN apt update
```

Suppose tomorrow you want to build using Ubuntu 22.04 instead.

You would have to edit the Dockerfile:

```dockerfile
FROM ubuntu:22.04
```

Instead, make it dynamic:

```dockerfile
ARG VERSION=24.04

FROM ubuntu:${VERSION}
```

Now you can choose the version during the build.

---

# Syntax

## Declare an argument

```dockerfile
ARG VERSION
```

---

## Declare with a default value

```dockerfile
ARG VERSION=24.04
```

If no value is supplied, Docker uses `24.04`.

---

# Passing an ARG during build

Dockerfile:

```dockerfile
ARG VERSION=24.04

FROM ubuntu:${VERSION}
```

Build:

```bash
docker build -t myimage .
```

Docker uses:

```text
24.04
```

Override it:

```bash
docker build --build-arg VERSION=22.04 -t myimage .
```

Docker now uses:

```text
22.04
```

---

# Another Example

```dockerfile
FROM python:3.12

ARG APP_VERSION=1.0

RUN echo "Application Version: ${APP_VERSION}"
```

Build:

```bash
docker build \
--build-arg APP_VERSION=2.1 \
-t python-app .
```

Output:

```text
Application Version: 2.1
```

---

# Where can `ARG` be used?

Example:

```dockerfile
FROM ubuntu

ARG USERNAME=venkat

RUN echo $USERNAME

RUN mkdir /home/$USERNAME
```

Docker substitutes the value during the build.

---

# `ARG` Before `FROM`

One special feature is that `ARG` can appear before `FROM`.

```dockerfile
ARG VERSION=24.04

FROM ubuntu:${VERSION}
```

This lets you choose the base image version at build time.

---

# Scope of `ARG`

`ARG` exists **only during the build stage where it is declared**.

Example:

```dockerfile
ARG VERSION=24.04

FROM ubuntu:${VERSION}

RUN echo $VERSION
```

Works because `ARG` is used to construct the `FROM` line.

If you want to use `VERSION` after `FROM`, declare it again:

```dockerfile
ARG VERSION=24.04

FROM ubuntu:${VERSION}

ARG VERSION

RUN echo $VERSION
```

---

# `ARG` vs `ENV`

This is one of the most common interview questions.

|Feature|ARG|ENV|
|---|---|---|
|Available during build|✅ Yes|✅ Yes|
|Available when container runs|❌ No|✅ Yes|
|Can be overridden at build time|✅ Yes (`--build-arg`)|❌ Not with `--build-arg`|
|Stored in final image configuration|Not intended for runtime use|✅ Yes|

Example:

```dockerfile
ARG APP_VERSION=1.0

ENV APP_VERSION=${APP_VERSION}
```

Now:

```bash
docker build \
--build-arg APP_VERSION=2.0 \
-t app .
```

Inside the running container:

```bash
echo $APP_VERSION
```

Output:

```text
2.0
```

Here, `ARG` passes the value at build time, and `ENV` makes it available at runtime.

---

# Real-world Example

Imagine you have three environments.

Development:

```text
Python 3.11
```

Testing:

```text
Python 3.12
```

Production:

```text
Python 3.13
```

Instead of maintaining three Dockerfiles:

```dockerfile
FROM python:3.11
```

```dockerfile
FROM python:3.12
```

```dockerfile
FROM python:3.13
```

Use one Dockerfile:

```dockerfile
ARG PYTHON_VERSION=3.12

FROM python:${PYTHON_VERSION}
```

Build different images:

```bash
docker build --build-arg PYTHON_VERSION=3.11 -t app-dev .
```

```bash
docker build --build-arg PYTHON_VERSION=3.12 -t app-test .
```

```bash
docker build --build-arg PYTHON_VERSION=3.13 -t app-prod .
```

---

# Does changing an `ARG` affect the build cache?

Yes.

If an `ARG` value changes and it's used by a Dockerfile instruction, the cache for that instruction and all following instructions is invalidated.

Example:

```dockerfile
FROM ubuntu

ARG VERSION

RUN echo $VERSION
```

Build:

```bash
docker build --build-arg VERSION=1 .
```

Later:

```bash
docker build --build-arg VERSION=2 .
```

The `RUN echo $VERSION` layer is rebuilt because its input changed.

---

# Best Practices

- Use `ARG` for values needed **only during the build**.
    
- Use `ENV` for values your application needs **while the container is running**.
    
- Provide sensible default values where appropriate.
    
- Use `ARG` to parameterize base image versions and build options.
    

> **Do not use `ARG` (or `ENV`) to pass secrets like passwords or API keys.** Build arguments can appear in image history and are not a secure way to handle sensitive data. Use Docker BuildKit secrets or another secret management solution instead.

---

# Common Interview Questions

### 1. What is `ARG` in Docker?

`ARG` defines build-time variables that can be passed using the `--build-arg` option during `docker build`.

---

### 2. What is the difference between `ARG` and `ENV`?

- `ARG` is available during the build only.
    
- `ENV` is available during both the build and when the container runs.
    

---

### 3. Can `ARG` be used in the `FROM` instruction?

Yes. Declare it before `FROM`:

```dockerfile
ARG VERSION=24.04
FROM ubuntu:${VERSION}
```

---

### 4. Can you change an `ARG` after the image is built?

No. `ARG` values are set during the build. To use a different value, you must rebuild the image.

---

### 5. Does changing an `ARG` invalidate the build cache?

Yes. If the `ARG` affects a Dockerfile instruction, that instruction and all following layers are rebuilt.

---

## One-line Interview Answer

> **`ARG` is a Dockerfile instruction used to define build-time variables. Values are passed with `--build-arg`, are available during image creation, can parameterize instructions like `FROM`, and are not intended to be runtime environment variables.**


__________________________________________________________________
__________________________________________________________________
# Docker Build Cache – Deep Dive

Docker Build Cache is one of the **most important Docker concepts** because it directly affects **build speed**, **CI/CD performance**, and **resource usage**. Interviewers often ask about it because it shows whether you understand how Docker builds images efficiently.

---

# What is Docker Build Cache?

When you run:

```bash
docker build -t myapp .
```

Docker **doesn't execute every instruction from scratch** every time.

Instead, it **stores the result of each Dockerfile instruction as a cached layer**.

On the next build, Docker checks:

> "Has anything changed since the last build?"

- If **No** → Reuse the cached layer.
    
- If **Yes** → Rebuild that layer and every layer after it.
    

Think of the cache as a collection of **saved checkpoints**.

---

# How Docker Builds an Image

Consider this Dockerfile:

```dockerfile
FROM ubuntu:24.04

RUN apt update

RUN apt install -y python3

COPY app.py /app/

CMD ["python3", "/app/app.py"]
```

Docker executes the Dockerfile **line by line**.

```
Step 1
FROM ubuntu
↓

Step 2
RUN apt update
↓

Step 3
RUN apt install python3
↓

Step 4
COPY app.py
↓

Step 5
CMD
```

Each step creates **a new image layer**.

```
Layer 5
CMD

Layer 4
COPY app.py

Layer 3
RUN apt install

Layer 2
RUN apt update

Layer 1
FROM ubuntu
```

Each of these layers is stored in Docker's cache.

---

# First Build

Imagine running:

```bash
docker build -t myapp .
```

Docker executes every instruction.

```
FROM ubuntu
↓

RUN apt update

↓

RUN apt install

↓

COPY app.py

↓

CMD
```

Suppose this takes:

```
FROM                  5 sec

apt update           30 sec

apt install          60 sec

COPY                  1 sec

CMD                   0 sec

----------------------------

Total = 96 seconds
```

Docker stores all five layers.

---

# Second Build (Nothing Changed)

You run:

```bash
docker build -t myapp .
```

again.

Docker checks every step.

```
FROM
✓ Cached

RUN apt update
✓ Cached

RUN apt install
✓ Cached

COPY app.py
✓ Cached

CMD
✓ Cached
```

Result:

```
Build completes in 1–2 seconds.
```

No commands are executed.

---

# Third Build (Only app.py Changed)

Suppose you edit:

```
app.py
```

Docker checks:

```
FROM ubuntu
✓ Same

RUN apt update
✓ Same

RUN apt install
✓ Same

COPY app.py
❌ Changed
```

Since the COPY changed:

```
COPY app.py
↓

CMD
```

are rebuilt.

Everything before COPY is reused.

---

# Why Docker Rebuilds Everything After a Changed Layer

Suppose:

```dockerfile
FROM ubuntu

RUN apt update

COPY app.py

RUN python setup.py install

CMD python app.py
```

If:

```
COPY app.py
```

changes,

Docker cannot know whether:

```
RUN python setup.py install
```

would produce the same result.

Therefore:

```
COPY
↓

RUN setup.py

↓

CMD
```

are rebuilt.

This is called **cache invalidation**.

---

# Cache Invalidation Rule

A simple rule to remember:

> **Once one layer changes, every layer after it must be rebuilt.**

Example:

```
Layer1
✓

Layer2
✓

Layer3
✓

Layer4
Changed

Layer5
Rebuild

Layer6
Rebuild
```

---

# Example with Python

Bad Dockerfile:

```dockerfile
FROM python:3.12

COPY . .

RUN pip install -r requirements.txt

CMD ["python","app.py"]
```

Problem:

Whenever **any file changes**, Docker must:

```
COPY
↓

pip install
↓

CMD
```

That means dependencies are installed every build.

---

Better Dockerfile

```dockerfile
FROM python:3.12

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY . .

CMD ["python","app.py"]
```

Now:

Changing:

```
app.py
```

does NOT reinstall packages.

Docker reuses:

```
COPY requirements.txt
↓

RUN pip install
```

Only:

```
COPY app.py

CMD
```

are rebuilt.

Huge speed improvement.

---

# Example with Node.js

Poor Dockerfile

```dockerfile
FROM node:22

COPY . .

RUN npm install

CMD ["npm","start"]
```

Every code change triggers:

```
npm install
```

Again.

---

Optimized Dockerfile

```dockerfile
FROM node:22

COPY package*.json ./

RUN npm install

COPY . .

CMD ["npm","start"]
```

Now:

Only if:

```
package.json
```

changes,

Docker reruns:

```
npm install
```

Otherwise it's cached.

---

# Build Context Matters

When you run:

```bash
docker build .
```

Docker first sends the **build context** (the files in the current directory, excluding what's in `.dockerignore`) to the Docker daemon.

If unnecessary files change, they can invalidate `COPY` instructions.

Use a `.dockerignore` file to exclude things like:

```
.git
node_modules
*.log
dist
__pycache__
```

This keeps the context smaller and improves cache reuse.

---

# Build Without Cache

Sometimes you want a completely fresh build.

```bash
docker build --no-cache -t myapp .
```

Docker ignores every cached layer.

Everything executes again.

Useful when:

- dependencies are corrupted
    
- debugging
    
- verifying reproducible builds
    

---

# Where Does Docker Store Build Cache?

Docker stores cached layers locally in its data directory (managed by Docker, often under `/var/lib/docker` on Linux with the default storage driver). You don't manually manage individual cache files.

Check cache usage:

```bash
docker system df
```

Example:

```
Build Cache

Size

9 GB
```

---

# Remove Build Cache

```bash
docker builder prune
```

Remove all unused cache:

```bash
docker builder prune -a
```

---

# Does Removing Build Cache Affect Containers?

No.

```
Running Containers
↓

No impact
```

```
Existing Images
↓

No impact
```

```
Future Builds
↓

Slower until cache is rebuilt
```

---

# Multi-Stage Builds and Cache

Each stage in a multi-stage Dockerfile has its own cache.

Example:

```dockerfile
FROM node:22 AS builder
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```

If only your application code changes, Docker can often reuse the cached dependency installation layer in the builder stage, making rebuilds much faster.

---

# Best Practices

- Put instructions that change **least often** (like installing OS packages or app dependencies) before instructions that change **frequently** (like copying source code).
    
- Copy dependency files (`requirements.txt`, `package.json`, `pom.xml`) before the rest of the source.
    
- Use `.dockerignore` to reduce the build context.
    
- Combine related `RUN` commands where appropriate to keep images efficient.
    
- Use multi-stage builds to produce smaller, cleaner images.
    
- Use `--no-cache` only when you genuinely need a fresh build.
    

---

# Common Interview Questions

### 1. What is Docker Build Cache?

It stores intermediate image layers so Docker can reuse unchanged layers during future builds.

---

### 2. When is the cache invalidated?

When a Dockerfile instruction changes or the inputs to that instruction change (for example, files copied with `COPY`, build arguments, or the base image).

---

### 3. Why is Dockerfile instruction order important?

Because Docker reuses cached layers from top to bottom. Placing stable instructions before frequently changing ones maximizes cache reuse.

---

### 4. What happens after one layer changes?

That layer and all subsequent layers are rebuilt.

---

### 5. Does deleting the build cache affect running containers?

No. It only removes cached build layers. Existing images and running containers continue to work normally.

---

## Simple Analogy

Imagine writing a 100-page report.

- The **first time**, you write all 100 pages.
    
- The **next time**, if only **page 95** changes, you don't rewrite pages 1–94.
    
- You reuse the unchanged pages and update only the affected part.
    

Docker Build Cache works in a similar way: it reuses unchanged layers instead of rebuilding everything, which makes image builds much faster.

Docker build cache is a **very common interview topic**, especially for DevOps, Docker, and CI/CD roles. Interviewers want to know whether you understand **how Docker optimizes image builds**.

Here are some of the most frequently asked questions.

---

## 1. What is Docker build cache?

**Answer:**

Docker build cache stores intermediate image layers created during `docker build`. When rebuilding an image, Docker reuses unchanged layers instead of executing every instruction again, making builds much faster.

---

## 2. How does Docker decide whether to use the cache?

**Answer:**

Docker compares each Dockerfile instruction and its inputs with the previous build.

If nothing has changed, it reuses the cached layer.

If something changes, Docker rebuilds that layer and every layer after it.

**Example:**

```dockerfile
FROM python:3.12
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

If only your application code changes:

- ✅ `FROM` → cached
    
- ✅ `COPY requirements.txt` → cached
    
- ✅ `RUN pip install` → cached
    
- ❌ `COPY . .` → rebuilt
    

---

## 3. Why is Docker build cache important?

**Answer:**

It:

- Speeds up image builds.
    
- Reduces network downloads.
    
- Saves CPU usage.
    
- Makes CI/CD pipelines faster.
    

---

## 4. When is the cache invalidated?

**Answer:**

The cache is invalidated when:

- A Dockerfile instruction changes.
    
- Files used by `COPY` or `ADD` change.
    
- Build arguments (`ARG`) change.
    
- The base image changes (or is updated and pulled).
    

---

## 5. What happens after cache is invalidated?

Docker rebuilds:

- The changed layer.
    
- Every subsequent layer.
    

Docker cannot skip ahead and reuse later layers.

---

## 6. How can you avoid unnecessary rebuilds?

**Good Dockerfile:**

```dockerfile
FROM node:22

COPY package*.json ./
RUN npm install

COPY . .
```

Here, `npm install` runs only when `package.json` changes.

**Bad Dockerfile:**

```dockerfile
FROM node:22

COPY . .
RUN npm install
```

Now any source code change forces `npm install` to run again.

---

## 7. How do you build an image without using cache?

```bash
docker build --no-cache -t myapp .
```

Use this when you need a completely fresh build.

---

## 8. How do you remove the build cache?

```bash
docker builder prune
```

Remove all unused cache:

```bash
docker builder prune -a
```

---

## 9. Does deleting the build cache affect running containers?

**Answer:**

No.

It only removes cached build layers.

Running containers and existing images continue to work normally.

---

## 10. Difference between Image and Build Cache

|Image|Build Cache|
|---|---|
|Runnable artifact|Intermediate build layers|
|Used to create containers|Used to speed up builds|
|Required to run containers|Optional|
|Removed with `docker image rm`|Removed with `docker builder prune`|

---

## 11. What happens if you change the first line of a Dockerfile?

Example:

```dockerfile
FROM ubuntu:24.04
```

changed to

```dockerfile
FROM ubuntu:26.04
```

**Answer:**

Docker rebuilds **every layer**, because all subsequent layers depend on the base image.

---

## 12. Why is instruction order important?

The order determines how effectively Docker can reuse cached layers.

Place instructions that change **less frequently** (like installing dependencies) before instructions that change **often** (like copying application code). This maximizes cache reuse and speeds up builds.

---

# Scenario-based interview question

**Interviewer:** _"Your Docker image takes 15 minutes to build every time, even when only one source file changes. How would you optimize it?"_

**Expected answer:**

- Check the Dockerfile order.
    
- Copy dependency files (`package.json`, `requirements.txt`, `pom.xml`) first.
    
- Install dependencies.
    
- Copy application code last.
    
- Use a `.dockerignore` file to avoid sending unnecessary files to the build context.
    
- Consider multi-stage builds to keep images smaller and builds cleaner.
    

---

## One-line interview definition

> **Docker build cache is a mechanism that stores intermediate image layers so Docker can reuse unchanged layers during subsequent builds, reducing build time and improving CI/CD efficiency.**


# docker top 

### When do we use `docker top`?

- See what processes are running inside a container.
    
- Verify whether your application is actually running.
    
- Troubleshoot a container that isn't behaving as expected.
    
- Check the PID and command of running processes.


Let's go through each point with a simple example.

### 1. See what processes are running inside a container

Suppose you start an Nginx container:

```bash
docker run -d --name my-nginx nginx
```

Now run:

```bash
docker top my-nginx
```

Output:

```text
UID   PID   CMD
root  1234  nginx: master process nginx -g daemon off;
101   1235  nginx: worker process
```

This tells you that **Nginx is running**, and it has a master process and a worker process.

**Think of it like:** Opening Windows Task Manager to see which programs are running.

---

### 2. Verify whether your application is actually running

Imagine you started a Java application:

```bash
docker run -d --name my-app my-java-app
```

Run:

```bash
docker top my-app
```

If you see:

```text
java -jar app.jar
```

your application is running.

If you don't see the Java process, the application may have crashed or exited.

---

### 3. Troubleshoot a container that isn't behaving as expected

Suppose your application is responding very slowly.

You check:

```bash
docker top my-app
```

Instead of seeing:

```text
java -jar app.jar
```

you see:

```text
sleep 1000
```

This means someone started the container with a `sleep` command instead of the application.

Or perhaps you expect two processes (an application and a helper process), but only one is running. `docker top` helps you identify that.

---

### 4. Check the PID and command of running processes

Example output:

```text
UID   PID   CMD
root  4567  python app.py
```

Here:

- **PID (Process ID):** `4567`
    
- **Command:** `python app.py`
    

The PID helps when debugging. For example, if you're investigating resource usage, you can identify the exact process responsible.

---

### Interview scenario

**Interviewer:** _"Your container is running, but your application isn't accessible. What would you check?"_

A good answer is:

1. `docker ps` → Is the container running?
    
2. `docker logs <container>` → Are there any application errors?
    
3. `docker top <container>` → Is the expected application process (such as `java`, `node`, or `nginx`) actually running?
    
4. `docker exec -it <container> sh` → Log into the container for further investigation.
    

This sequence shows a practical troubleshooting approach that interviewers often look for.


The key idea to understand is that a **Docker container is just one or more Linux processes running in isolation**.

When you run `docker top`, you're asking Docker:

> **"Show me the processes that are running inside this container."**

Let's use a real-world example.

### Example 1: Nginx container

You start an Nginx container:

```bash
docker run -d --name web nginx
```

Now imagine you run:

```bash
docker top web
```

You might see:

```text
UID   PID    CMD
root  1250   nginx: master process nginx -g daemon off;
101   1258   nginx: worker process
```

#### What does this tell you?

- There are **2 processes** running.
    
- The **master process** manages Nginx.
    
- The **worker process** serves incoming HTTP requests.
    

Without `docker top`, you only know the container is running. With `docker top`, you know **what is actually running inside it**.

---

## Example 2: Java application

Suppose your company has a Spring Boot application.

You start it:

```bash
docker run -d --name payment-app payment-service
```

Run:

```bash
docker top payment-app
```

Output:

```text
UID   PID    CMD
root  2251   java -jar payment-service.jar
```

This confirms your Java application is actually running.

Now imagine the output is:

```text
UID   PID    CMD
root  2251   sleep infinity
```

This tells you something is wrong.

The container is alive, but **your Java application never started**. Someone started the container with a `sleep` command instead.

---

## Example 3: Why not just use `docker ps`?

`docker ps` shows:

```text
CONTAINER ID   STATUS
abc123         Up 20 minutes
```

This only tells you:

> "The container exists and is running."

It **doesn't** tell you:

- Is Java running?
    
- Is Nginx running?
    
- Is Python running?
    
- Is the application stuck?
    

That's where `docker top` helps.

---

## Think of it like this

Imagine your house.

- **`docker ps`** is like standing outside the house and seeing the lights are on.
    
    - ✅ The house is occupied.
        
- **`docker top`** is like opening the door and seeing who is inside.
    
    - 👨 John is working.
        
    - 👩 Sarah is cooking.
        
    - 🧒 Kids are studying.
        

You now know **what is happening inside**, not just that the house is occupied.

---

## During troubleshooting

Suppose a user says:

> "The website is down."

You check:

```bash
docker ps
```

Output:

```text
payment-app   Up 2 hours
```

The container is running.

Next, you check:

```bash
docker top payment-app
```

Output:

```text
sleep infinity
```

Now you've found the problem:

- ✅ Container is running.
    
- ❌ The actual application process is **not** running.
    

This is exactly why `docker top` is useful—it helps you verify **whether the expected process inside the container is running**, rather than just confirming that the container itself is alive.


Error 3
--
If you get this error:

```text
$ docker top python-cont

Error response from daemon: container 77f17512518d8450f9caf3061426b09b6ae819083b16f48ca30c18bd642c2823 is not running
```

it means **`docker top` cannot show processes because the container has already stopped**. Your goal is to find **why** it stopped.

Here are the troubleshooting steps I would follow:

### Step 1: Check the container status

```bash
docker ps -a
```

Look for the **STATUS** column.

Example:

```text
CONTAINER ID   NAME          STATUS
77f17512518d   python-cont   Exited (1) 2 minutes ago
```

This confirms the container is not running.

---

### Step 2: Check the container logs (Most Important)

```bash
docker logs python-cont
```

This often tells you exactly why the container exited.

Examples:

```text
ModuleNotFoundError: No module named 'flask'
```

or

```text
python: can't open file 'app.py'
```

or

```text
Address already in use
```

---

### Step 3: Inspect the exit code

```bash
docker inspect python-cont
```

Or just retrieve the exit code:

```bash
docker inspect -f '{{.State.ExitCode}}' python-cont
```

Common exit codes:

- **0** → Application finished successfully.
    
- **1** → General application error.
    
- **125** → Docker failed to run the container.
    
- **126** → Command found but not executable.
    
- **127** → Command not found.
    
- **137** → Killed (often due to out-of-memory or `docker kill`).
    
- **143** → Gracefully stopped (`SIGTERM`).
    

---

### Step 4: Start the container again (if appropriate)

```bash
docker start python-cont
```

Then verify:

```bash
docker ps
```

If it's running, you can now use:

```bash
docker top python-cont
```

---

### Step 5: If it exits immediately again

Run it in the foreground to see the error directly:

```bash
docker start -a python-cont
```

Or recreate the container with:

```bash
docker run ...
```

This lets you watch the application's output in real time.

---

### Step 6: Inspect the container configuration

```bash
docker inspect python-cont
```

Check for:

- Image name
    
- Command (`Cmd`)
    
- Entrypoint
    
- Environment variables
    
- Mounted volumes
    
- Port mappings
    

A wrong command or missing volume can prevent the application from starting.

---

## Interview answer

> If `docker top` reports that the container is not running, I first check the container status using `docker ps -a`, then review the application logs with `docker logs`, inspect the exit code and configuration using `docker inspect`, restart the container if appropriate, and if it still exits immediately, I run it in the foreground to identify the root cause. Only after the container is running again would I use `docker top` to inspect its processes.

___

# docker inspect

When troubleshooting with `docker inspect`, you don't need to read the entire JSON. Focus on these sections—they solve most Docker issues.

|Field|Why it's important|Example issue|
|---|---|---|
|**State**|Checks if the container is running, exited, or restarting|Container exited immediately|
|**State.ExitCode**|Shows why the container stopped|`0` = success, `1` or other = error|
|**State.Error**|Displays Docker runtime errors|Mount or startup failures|
|**Config.Image**|Confirms which image the container was created from|Wrong image version|
|**Config.Cmd**|Shows the command executed|Incorrect startup command|
|**Config.Entrypoint**|Shows the entrypoint|Entrypoint overriding your command|
|**Config.Env**|Lists environment variables|Missing database credentials or API keys|
|**Mounts**|Shows mounted volumes and bind mounts|Data not persisting or wrong mount path|
|**NetworkSettings**|Displays IP address, networks, and ports|Network connectivity issues|
|**HostConfig.PortBindings**|Confirms published ports|Forgot `-p 8080:80`|
|**RestartPolicy**|Shows restart configuration|Container not restarting automatically|
|**HostConfig.Binds**|Lists host-to-container bind mounts|Wrong host directory mounted|
|**HostConfig.Memory / NanoCpus**|Shows resource limits|Container killed due to memory limits|
|**Created**|Shows when the container was created|Verify you're using the latest container|

### Most commonly checked during troubleshooting

```bash
docker inspect <container-name>
```

Look at these in order:

1. **State** → Is the container running?
    
2. **ExitCode** → Why did it stop?
    
3. **Error** → Any Docker runtime errors?
    
4. **Cmd & Entrypoint** → Is the correct command being executed?
    
5. **Environment Variables** → Are required variables present?
    
6. **Mounts** → Are volumes mounted correctly?
    
7. **PortBindings** → Are ports exposed and mapped?
    
8. **NetworkSettings** → Is the container on the correct network with the expected IP?
    

### Useful shortcuts

Instead of viewing the entire JSON, you can inspect specific fields:

```bash
docker inspect --format='{{.State.Status}}' <container-name>
```

```bash
docker inspect --format='{{.State.ExitCode}}' <container-name>
```

```bash
docker inspect --format='{{.Config.Cmd}}' <container-name>
```

```bash
docker inspect --format='{{.NetworkSettings.IPAddress}}' <container-name>
```

```bash
docker inspect --format='{{json .Mounts}}' <container-name>
```

### Interview Tip

If asked, _"How do you use `docker inspect` for troubleshooting?"_, a strong answer is:

> "I first check the container state, exit code, and any runtime errors. Then I verify the image, command, and entrypoint. Next, I confirm environment variables, mounted volumes, port mappings, and network settings to identify configuration or connectivity issues. This systematic approach helps isolate most container startup and runtime problems."

___
# ONBUILD

`ONBUILD` is a Dockerfile instruction that **registers a trigger**. The instruction is **not executed when the current image is built**. Instead, it is executed **automatically when another Dockerfile uses this image as its base image (`FROM`)**.

If you have:

```dockerfile
ONBUILD COPY . /app
```

it means:

> **When someone builds a new image using this image as the base, Docker will automatically execute `COPY . /app` during that build.**

### Example

**Parent Dockerfile**

```dockerfile
FROM python:3.12
ONBUILD COPY . /app
```

Build it:

```bash
docker build -t python-base .
```

Nothing is copied into `/app` at this stage.

---

**Child Dockerfile**

```dockerfile
FROM python-base
RUN ls /app
```

When you build the child image:

```bash
docker build -t myapp .
```

Docker automatically runs:

```dockerfile
COPY . /app
```

before executing:

```dockerfile
RUN ls /app
```

### Use Case

`ONBUILD COPY . /app` is useful when creating **base images** that should automatically copy an application's source code for all child images.

### Interview Answer

> **`ONBUILD COPY . /app` registers a build trigger. It does not execute while building the current image; instead, it automatically copies the child project's files into `/app` when another image is built using this image as its base.**

____

# Dockerfile Instruction: `ENV` – Deep Dive

## What is `ENV`?

`ENV` stands for **Environment Variable**.

It is used to define environment variables that are available:

- During the image build (after the `ENV` instruction)
    
- Inside the running container
    

Unlike `ARG`, `ENV` values remain available when the container starts.

# Why do we need `ENV`?

Suppose your application needs:

- Database URL
    
- Application mode
    
- Port number
    
- Timezone
    

Instead of hardcoding these values inside your application, you can define them using `ENV`.

Example:

```dockerfile
FROM python:3.12

ENV APP_ENV=production

CMD ["python", "app.py"]
```

Now, inside the running container:

```bash
echo $APP_ENV
```

Output:

```text
production
```

---

# Syntax

## Single Variable

```dockerfile
ENV APP_ENV=production
```

---

## Multiple Variables

```dockerfile
ENV APP_ENV=production \
    PORT=8080 \
    TZ=Asia/Kolkata
```

---

# Example

```dockerfile
FROM ubuntu

ENV NAME=Venkat

RUN echo $NAME

CMD ["bash"]
```

During image build:

```text
Venkat
```

Run the container:

```bash
docker run -it myimage bash
```

Inside the container:

```bash
echo $NAME
```

Output:

```text
Venkat
```

The variable is still available because `ENV` persists into the container.

---

# Using `ENV` in Dockerfile Instructions

```dockerfile
FROM ubuntu

ENV APP_HOME=/app

WORKDIR $APP_HOME

COPY . .

CMD ["bash"]
```

Docker substitutes:

```text
WORKDIR /app
```

---

# Overriding `ENV` at Runtime

Dockerfile:

```dockerfile
ENV PORT=8080
```

Run normally:

```bash
docker run myimage
```

Inside container:

```text
PORT=8080
```

Override it:

```bash
docker run -e PORT=9090 myimage
```

Now:

```bash
echo $PORT
```

Output:

```text
9090
```

The runtime value overrides the Dockerfile value.

---

# `ENV` vs `ARG`

|Feature|ENV|ARG|
|---|---|---|
|Available during build|✅ Yes|✅ Yes|
|Available in running container|✅ Yes|❌ No|
|Can be overridden at build time|❌ No|✅ Yes (`--build-arg`)|
|Can be overridden at runtime|✅ Yes (`docker run -e`)|❌ No|
|Used for runtime configuration|✅ Yes|❌ No|

---

# `ARG` + `ENV` Together

```dockerfile
ARG APP_VERSION=1.0

ENV APP_VERSION=${APP_VERSION}
```

Build:

```bash
docker build --build-arg APP_VERSION=2.0 -t myapp .
```

Run:

```bash
docker run myapp
```

Inside container:

```bash
echo $APP_VERSION
```

Output:

```text
2.0
```

Here:

- `ARG` provides the value during the build.
    
- `ENV` stores it for runtime.
    

---

# Common Use Cases

### Application Mode

```dockerfile
ENV APP_ENV=production
```

---

### Port

```dockerfile
ENV PORT=8080
```

---

### Timezone

```dockerfile
ENV TZ=Asia/Kolkata
```

---

### Java

```dockerfile
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk
```

---

### Python

```dockerfile
ENV PYTHONUNBUFFERED=1
```

---

### Node.js

```dockerfile
ENV NODE_ENV=production
```

---

# Does Changing `ENV` Affect Build Cache?

Yes.

Example:

```dockerfile
ENV APP_ENV=production

RUN echo $APP_ENV
```

Later:

```dockerfile
ENV APP_ENV=development
```

Since the environment variable changed, Docker rebuilds that layer and all following layers.

---

# Inspecting Environment Variables

To see the environment variables stored in an image:

```bash
docker inspect myimage
```

Look for:

```text
Config
   Env
```

Or inside a running container:

```bash
docker exec -it mycontainer env
```

or

```bash
docker exec -it mycontainer printenv
```

---

# Best Practices

- Use `ENV` for values needed while the container is running.
    
- Group related environment variables together.
    
- Allow runtime overrides with `docker run -e` when appropriate.
    
- Avoid storing passwords, API keys, or other secrets in `ENV`, because they can be viewed through `docker inspect` or from inside the container.
    

---

# Common Interview Questions

### 1. What is `ENV` in Docker?

`ENV` sets environment variables that are available during image build (after the instruction) and inside the running container.

---

### 2. What is the difference between `ENV` and `ARG`?

- `ENV` is available during build and runtime.
    
- `ARG` is available only during image build.
    

---

### 3. Can you override an `ENV` variable?

Yes.

```bash
docker run -e PORT=9090 myimage
```

---

### 4. Does changing an `ENV` invalidate the build cache?

Yes.

If an `ENV` value changes, that layer and all following layers are rebuilt.

---

### 5. Can `ENV` be used in other Dockerfile instructions?

Yes.

It can be referenced in instructions such as `WORKDIR`, `RUN`, `CMD`, and `ENTRYPOINT` using variable substitution.

---

# Interview Answer (One Line)

> **`ENV` is a Dockerfile instruction used to define environment variables that are available during image build and inside the running container. These variables can also be overridden at runtime using `docker run -e`.**

___________________________
### **ENTRYPOINT – Explanation**

`ENTRYPOINT` is used to specify the **main executable (primary application)** that a container should run when it starts.

In simple words, `ENTRYPOINT` tells Docker:

> **"This container is built to run this specific application."**

For example:

```dockerfile
FROM ubuntu

ENTRYPOINT ["ping"]
```

This image is designed to run the `ping` command whenever the container starts.

If you run:

```bash
docker run myimage google.com
```

Docker automatically executes:

```bash
ping google.com
```

Here:

- `ping` comes from `ENTRYPOINT`.
    
- `google.com` is passed as an argument to `ping`.
    

### Why is `ENTRYPOINT` used?

It ensures that every time someone starts the container, the intended application always runs.

Examples:

- An Nginx container starts the Nginx web server.
    
- A MySQL container starts the MySQL database server.
    
- A Redis container starts the Redis server.
    
- A Java application container starts the Java application.
    

### Key Point

Think of `ENTRYPOINT` as the **fixed command** that defines the purpose of the container. Users can provide additional arguments when running the container, but the primary executable remains the same unless `--entrypoint` is explicitly used to override it.

### Interview Answer

> **`ENTRYPOINT` specifies the primary executable that always runs when the container starts. It defines the main purpose of the container, while allowing users to pass additional command-line arguments to that executable.**

______
### **CMD – Explanation**

`CMD` is used to specify the **default command or default arguments** that Docker runs when a container starts.

In simple words, `CMD` tells Docker:

> **"If the user doesn't provide a command while starting the container, run this command by default."**

For example:

```dockerfile
FROM ubuntu

CMD ["echo", "Hello, Docker!"]
```

If you run:

```bash
docker run myimage
```

Docker executes:

```bash
echo "Hello, Docker!"
```

Output:

```text
Hello, Docker!
```

---

### Can `CMD` be overridden?

Yes.

If the user provides a command while starting the container, Docker ignores the `CMD` instruction.

Example:

Dockerfile:

```dockerfile
FROM ubuntu

CMD ["echo", "Hello, Docker!"]
```

Run:

```bash
docker run myimage ls
```

Docker executes:

```bash
ls
```

instead of:

```bash
echo "Hello, Docker!"
```

---

### Why is `CMD` used?

It provides a **default behavior** for the container while still allowing users to override it when needed.

Examples:

- A Python image may use:
    
    ```dockerfile
    CMD ["python3"]
    ```
    
- An Ubuntu image may use:
    
    ```dockerfile
    CMD ["bash"]
    ```
    
- A web application image may use:
    
    ```dockerfile
    CMD ["python", "app.py"]
    ```
    

---

### Key Point

Think of `CMD` as the **default command** for the container. It is flexible because users can replace it by specifying another command in `docker run`.

---

### Interview Answer

> **`CMD` specifies the default command or arguments that Docker runs when a container starts. If the user provides a command during `docker run`, it overrides the `CMD` instruction.**