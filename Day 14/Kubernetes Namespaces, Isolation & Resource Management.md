##### Table of Contents
- What are Namespaces in Kubernetes?
- Analogy to Understand Namespaces
- Why Use Namespaces?
- Default Namespace in Kubernetes
- Working with Namespaces
	- Imperative and Declarative Creation
	- Commands to List, Create, and Delete Namespaces
	- Use `-n, -A`, and `--all-namespace` Flags
- Deploying Frontend and Backend in a Namespace
- Testing Namespace Isolation
- Setting a Default Namespace in the Kubernetes Context
- Best Practices for Using Namespaces
- Summary
- References
---
###### What are Namespaces in Kubernetes?
A namespace in Kubernetes is a logical isolation/partition within a cluster that helps organize and isolate resources.

Namespaces enable:
- **Isolation & Security:** Separate workloads to prevent unwanted interactions.
- **Avoiding Naming Conflicts:** Resources with the same name can exist in different namespaces.
- **Resource Management:** Apply resource quotas and limits at the namespace level.
- **Application Segregation:** Separate environments (e.g., dev, test, prod) or projects.
- **Organizational Clarity:** Manage resources by teams, departments, or projects.
----
###### Analogy to Understand Namespaces

Image a large house where four families live together.

- Without namespace, all families share common spaces, leading to no privacy, security, or organization.
- When you create rooms for each family, each family has its own space, improving isolation, security, and organization.

###### Relating to Kubernetes:

- The large house is the Kubernetes cluster.
- The families are applications or workloads.
- Rooms are namespaces, providing segregation and control over resources.
-----
Why Use Namespaces?
- 🛡️**Security:** Limit access and apply network policies.
- 🚦**Resource Management:** Define quotas and limits for CPU and memory.
- 🎯**Environment Management:** Isolate development, testing, and production environments.
- 📦**Multi-Tenancy:** Host multiple applications in the same cluster without conflicts.
- 🧹**Simplification:** Manage related resources together.
----
###### Default Namespace in Kubernetes
When you run kubectl get namespace, you'll see these default namespaces.

| Namespace       | Purpose                                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| default         | Kubernetes include this namespace so that you can start using your new cluster without first creating a namespace                                |
| kube-system     | The namespace for objects created by the Kubernetes system. Holds Kubernetes control plane components (e.g., kube-dns, kube-proxy).              |
| kube-publis     | This namespace is readable by all clients (including those not authenticated). Publicly accessible data, primarily used for cluster information. |
| kube-node-lease | Heartbeats of nodes in the cluster, used by the control plan for node health.                                                                    |
Note: In KIND (Kubernetes IN Docker) cluster, the local-path-storage namespace is created by default to support persistent storage using the local Path Provisioner.

----
###### Working with Namespaces

Creating Namespaces
1. Imperative Way:
	`kubectl create namespace app1-ns`

2. Declarative Way:
```
	#app1-ns.yaml
	apiVersion: v1
	kind: Namespace
	metadata:
		name: app1-ns
```
```
	kubectl apply -f app1-ns.yaml
```
-----
###### Viewing and Deleting Namespaces

View All Namespaces:
```
kubectl get namespace
```

To check current namespace:
```kubectl config view --minify --output 'jsonpath={..namespace}'```


To switch to another Namespace:
```
kubectl config set-context --current -n <namespace-name>
```

Delete a Namespace:
```
kubectl delete namespace app1-ns
```

**Warning:** Deleting a namespace removes all resources within it, including pods, services, configmaps, secrets etc.

-----
###### Using Namespace Flags

| Flag                             | Description                                 |
| -------------------------------- | ------------------------------------------- |
| -n namespace-name or --namespace | Execute commands within a spefic namespace. |
| -A or --all-namespaces           | Execute commands across all namespaces.     |
```
kubectl get pods -n app1-ns
kubectl get services -A
```

###### Deploying Frontend and Backend in a Namespace

- We'll use existing YAML files for our frontend and backend deployments.
- Deploying the frontend and backend components in the app1-ns namespace.
-----
Best Practices for Using Namespaces

- **Segregate Workloads:** Separate dev, test, and prod environments.
- **Use Namespaces for Multi-Tenancy:** Avoid resource conflicts by isolating teams or projects.
- **Resource Quotas:** Set limits on CPU, memory, and storage per namespace.
- **Namespace Naming Conventions:** Use clear and consistent names (e.g., `team-app-env` → `frontend-prod-ns`).
- **Avoid Manual Deletion:** Use `kubectl delete namespace` carefully, as it removes all resources within.
- **Apply Network Policies:** Secure namespace using network policies to control traffic flow.
----
