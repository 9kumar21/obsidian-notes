When a container is **running** but the application is **not accessible from `localhost`**, follow this troubleshooting flow. This is exactly the kind of approach interviewers expect.

---

# Docker Application Not Accessible - Troubleshooting Checklist

## Step 1: Check if the container is actually running

```bash
docker ps
```

Verify:

- Container status is `Up`
    
- Correct container name
    
- Port mapping is correct
    

Example:

```text
PORTS
0.0.0.0:8080->5000/tcp
```

If the container exited:

```bash
docker ps -a
```

Then check logs.

---

## Step 2: Check Port Mapping

```bash
docker port python-cont
```

Expected:

```text
5000/tcp -> 0.0.0.0:8080
```

If nothing appears, recreate the container.

---

## Step 3: Verify the application is listening inside the container

Open a shell:

```bash
docker exec -it python-cont sh
```

or

```bash
docker exec -it python-cont bash
```

Now check listening ports.

Alpine:

```bash
netstat -tuln

netstat -ano | findstr :5000
```

Ubuntu:

```bash
ss -tuln
```

Expected:

```
0.0.0.0:5000
```

Not:

```
127.0.0.1:5000
```

---

## Step 4: Verify the application itself

Inside the container:

```bash
curl localhost:5000
```

If this fails:

The application itself is not running correctly.

---

## Step 5: Check application binding

This is the **most common issue**.

Many Python Flask apps start like this:

```python
app.run()
```

Default:

```
127.0.0.1
```

Docker cannot expose this.

Correct:

```python
app.run(host="0.0.0.0", port=5000)
```

Without `0.0.0.0`, Docker port publishing won't work.

---

## Step 6: Verify Dockerfile EXPOSE

```dockerfile
EXPOSE 5000
```

Not mandatory, but helps documentation.

---

## Step 7: Verify the run command

If the app listens on port 5000:

```bash
docker run -d -p 8080:5000 python-img
```

Notice:

```
Host:Container
8080 : 5000
```

Your command was:

```bash
docker run -d -p 8080:500
```

This maps **host 8080 → container 500**, which is likely incorrect if your application listens on **5000**.

---

## Step 8: Inspect the container

```bash
docker inspect python-cont
```

Check:

- PortBindings
    
- IP Address
    
- Running state
    
- Network
    

---

## Step 9: Test from inside the container

```bash
docker exec -it python-cont curl localhost:5000
```

Works?

- Yes → Docker networking/port mapping issue.
    
- No → Application issue.
    

---

## Step 10: Test using the container IP

Find IP:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' python-cont
```

Example:

```
172.17.0.2
```

Test:

```bash
curl 172.17.0.2:5000
```

If this works but localhost doesn't:

The issue is with port publishing.

---

## Step 11: Check host port conflicts

Maybe another process is already using 8080.

Linux:

```bash
sudo lsof -i :8080
```

Windows:

```cmd
netstat -ano | findstr 8080
```

---

## Step 12: Check firewall/security software

Sometimes:

- Windows Defender Firewall
    
- Antivirus
    
- Corporate VPN
    

can block the port.

---

# Common Root Causes

|Problem|Solution|
|---|---|
|Wrong port mapping|`-p 8080:5000`|
|App listening on `127.0.0.1`|Bind to `0.0.0.0`|
|Wrong application port|Verify with `netstat`/`ss`|
|App never started|Check logs and `curl localhost` inside container|
|Host port already used|Use another host port|
|Container exited|`docker ps -a` and inspect logs|
|Firewall blocking|Allow the port|

---

# Interview Answer (30 seconds)

> "If a Docker container is running but the application isn't accessible, I first verify the container is up using `docker ps`, then check the port mapping with `docker port`. Next, I enter the container using `docker exec` and confirm the application is listening on the expected port and bound to `0.0.0.0` rather than `127.0.0.1`. I test the application locally inside the container using `curl localhost:<port>`, inspect the container configuration with `docker inspect`, verify there are no host port conflicts, and finally check firewall or networking issues if everything else looks correct."