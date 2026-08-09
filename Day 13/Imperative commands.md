##### Table of Contents

Why are Imperative Commands Important?
Imperative Commands for Backend Deployment and Service
Imperative Commands for Frontend Deployment and Service
Creating Test Pods Imperatively
Key Takeaways

Why are Imperative Commands Important?
Imperative commands are crucial for:
- ***Daily Troubleshooting:*** Quicky create pods or services to check **connectivity** or **network** **policies**.
- ***CKA Exam:*** Saves time when you need to quickly generate *YAML* manifests using -o yaml > file.yaml.
- **Real-World Scenarios:** Useful when testing **accessibility**, **deployment** **temporary** **resources**, or **verifying** **configurations**.

##### Important Flags to Know:
-  `--dry-run=client:` Validates the command locally without contacting the API server. No resources are created.
- `--dry-run=server:` Validates the command against the API server, ensuring schema and admission control checks are performed.
- `--rm:` Deletes the pod as soon as you exit the container, ideal for temporary pods used in troubleshooting.
- -------
  
##### Imperative Commands for Backend Deployment and Service
##### Step 1: Create Backend Deployment
```
kubectl create deployment backend-deploy --image=hashicorp/http-echo --replicas=3 --port=5678 --dry-run=client -o yaml > backend-deploy.yaml
```
- `--image:` Specifies the **container image** (hashicorp/http-echo).
- `--replicas:` Defines 3 **replicas** for **high** **availability**.
- `--port:` The **port** **on** which the **container** **listens** (5678).
- `--dry-run=client:` Validates the command without contacting the API server. No resources are created (because of the -o yaml part) to backend-deploy.yaml.

##### Step 2: Create Backend Service

`kubectl expose deployment backend-deploy --type=ClusterIP --port=9090 --target-port=5678 --name=backend-svc --dry-run=client -o yaml >> backend-deploy.yaml`

- `--type=ClusterIP:` Creates an internal-only service.
- `--port:` Exposes port 9090 on the service.
- `--target-port:` Redirects traffic to port 5678 on the container.
- `>>:` Appends the service YAML to backend-deploy.yaml.
##### Manual Steps:
- Add the YAML separator (---) to separate the deployment and service objects.
- Edit the labels as needed.
 `args` field to set the response text ("Hello from Backend").
 `args:`
    - "-text=Hello from Backend"
