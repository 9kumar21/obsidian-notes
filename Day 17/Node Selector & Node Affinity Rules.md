###### Table Contents
- Introduction
- Node Placement Strategies
- Understanding Node Selector
- Limitations of Node Selector
- Cluster Setup for Demonstration
- Demonstration: Node Selector
- Key Takeaways for `nodeSelector`
- Node Affinity in Kubernetes
- Demo: Node Affinity
- OR Condition (Multiple Label Matches)
- AND Condition (Multiple Label Matches)
- Node Affinity Operators
- Node Anti-Affinity
- Summary
- Understand `preferDuringSchedulingIgnoreDuringExecution`
- Demonstration: Using Preferred Node Affinity
- Key Considerations
- Node Affinity, `nodeSelector`, and Taints & Tolerations
- References
----
###### Introduction

In K8s, scheduling decisions determine where a pod runs within the cluster. While K8s has a built-in **scheduler** that automatically assigns pods to nodes, administrators often need to influence **scheduling decisions** to ensure specific workloads run on designated nodes.

To achieve this, K8s provides label-based scheduling techniques such as:

- **Node Selector** (Basic)
- **Node Affinity** and **Anti-Affinity** (Advanced)

This session focuses on **Node Selector**, its **advantages**, **limitations**, and how it compares to Node Affinity.

----
###### Node Placement Strategies

Pods can be scheduled onto specific nodes using labels and selectors. This ensures:

- Workloads are placed on nodes with required hardware (e.g., GPUs, SSDs)
- Applications are isolated by environments (e.g., prod, staging, production).
- Compliance with resource constrains (e.g., memory, CPU).

To implement this, K8s offers two main approaches:

1. **Node Selecto**r - Assigns pods to nodes based on simple key-value labels.
2. **Node Affinity** - Provides more flexibility, including soft preferences and multiple conditions.

###### How It Works

1. Assign labels to nodes
```
kubectl label node my-first-cluster-worker <label-key>=<label-value>
```

2. Define a `nodeSelector` in the pod spec
```
   nodeSelector:
	   key: value
```

3. Pods are scheduled only on nodes matching the label
----
###### Limitations of Node Selector

| Limitations      | Explanation                                                                 |
| ---------------- | --------------------------------------------------------------------------- |
| Strict Placement | If no node matches the label, the pod remains in the Pending state.         |
| No Preferences   | It does not allow "soft" preferences--either a node matches or it does not. |
| No OR Condition  | You cannot specify "schedule on nodes with `storage=ssd` OR `storage=hdd`"  |
For more advances scheduling needs, Node Affinity provides a more flexible alternative.

------
###### Cluster Setup for Demonstration

Before we begin our demonstration of **Node Selectors, Node Affinity and Node Anti-Affinity**, let's first review our Kubernetes cluster setup. We have a total of **three nodes** in our KIND cluster:

1. **my-second-cluster-control-plane** → This is the **control-plane node** responsible for managing the cluster.
2. **my-second-cluster-worker** → This is a **worker node** where application workloads can be scheduled.
3. **my-second-cluster-worker2** → This is another **worker node** available for scheduling workloads.
-----
###### Demonstration: Node Selector

Step1: Check Existing Labels on Nodes
```
kubectl get nodes --show-labels
kubectl describe node my-second-cluster-worker
```
This will display all the labels assigned to the nodes.

----
Step 2: Labels a Node
```
kubectl label node my-first-cluster-worker storage=ssd
```

This assigns the label `storage=ssd` to `my-first-cluster-worker.`

----
Step 3: Deploy a Pod with `nodeSelector`
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ns-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
	    nodeSelector: 
		    storage: ssd
		container:
		- name: nginx
		  image: nginx
```

- Pods will only be scheduled on my-second-cluster-worker because it has the label storage=ssd.
- If no nodes has storage=ssd, the pods remain in the Pending state.
----
Step 4: Add Multiple Labels to a Node

A node can have multiple labels. A Pod will only be scheduled if all labels in `nodeSelector` match.

```
kubectl label nodes my-first-cluster-worker env=prod
```
Modify the pod spec to require multiple labels:
```
nodeSelector
	storage: ssd
	env: prod
```

✔️ The pod will only be scheduled if a node has both storage=ssd and env=prod.
❌ If no such node exists, the pod will remain in the Pending state.

----
Step 5: Removing Labels

To remove a label from a node:

```
kubectl label nodes my-first-cluster-worker storage-
```
This removes the `storage=ssd` label.

After this, any pods with `nodeSelector`: { `storage: ssd` } will not be scheduled.

-----
###### Key Takeaways for `nodeSelector`

✔️ Node Selector is a simple, strict way to assign pods to nodes.
✔️ Pods are only scheduled if a node has all required labels.
✔️Lack of flexibility makes it unsuitable for complex scheduling needs.
✔️Node Affinity is the advanced alternative with more features.

[[K8s VarJosh & Me/Day 17/Deep Dive#`NodeSelector`|Deep Dive]]

---
###### Node Affinity in Kubernetes

`nodeAffinity` is an advances version of `nodeSelector` that provides more flexibility in scheduling pods onto specific nodes.

- It allows hard constraints (must match) and soft preferences (prefer but not mandatory).
- It enables complex AND/OR conditions for node selection.``
- It supports set-based expressions (`In`, `NotIn`, `Exists`, etc.), unlike `nodeSelector`.

-------
###### Types of Node Affinity

| Type                                             | Behavior                                                                                                                      |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `requireDuringSchedulingIgnoreDuringExecution`   | **Hard rule** - The pod will only be scheduled on matching nodes. If no matching node exists, the pod stays in Pending state. |
| `preferredDuringSchedulingIgnoreDuringExecution` | **Soft rule** - The pod will prefer matching nodes but can be scheduled elsewhere if needed.                                  |
`IgnoredDuringExecution` means that **if the node's labels change after scheduling**, the pod will continue running on that node.

----
###### Demo: Node Affinity

Step 1: Label a Node
```
kubeclt label nodes my-first-cluster-worker storage=ssd
```
This assigns the label `storage=ssd` to `my-first-cluster-worker`.

---
Step 2: Deploy a Pod with Required Node Affinity
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: na-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage
                operator: In
                values:
                  - ssd
      containers:
        - name: nginx
          image: nginx
```

###### Expected Behavior

- Pods will only be scheduled on nodes with `storage=ssd`.
- If no node has `storage=ssd`, the pod remains in the Pending state.

This behaves similarly to `nodeSelector`, but `nodeAffinity` supports compiles conditions.

📌 More on Set-Based Selectors: [Day 10 - ReplicaSet](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2010#3-replicaset-rs)

-----
###### OR Condition (Multiple Label Matches)

If we want to allow scheduling on nodes with either `storage=ssd` or `storage=hdd`, we use `OR` logic with `In`.

Step 1: Label a First Node
```
kubectl label node my-first-cluster-worker storage=hdd
```

Step 2: Update Node Affinity to Allow Either SSD or HDD
```
affinity:
	nodeAffinity:
		requiredDuringSchedulingIngoreDuringExecution:
		nodeSelectorTerms:
		- matchExpressions:
		  - key: storage
		    operator: In
		    values:
			- ssd
			- hdd
```
**Expected Behavior**

- Pods can be scheduled on:
  ✅ my-first-cluster-worker (`storage=ssd`)
  ✅ my-first-cluster-worker2 (`storage=hdd`)
-----
**AND Condition (Multiple Label Requirements)**

To schedule only on nodes that have BOTH `storage=ssd` AND `env=prod`, we use multiple `matchExpressions`.

Step 1: Label a Node
```
kubectl label node my-first-cluster-worker env=pro
```

Step 2: Apply AND Condition
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage
          operator: In
          values:
            - ssd
        - key: env
          operator: In
          values:
            - prod
```
**Expected Behavior

- Pods will only be scheduled on nodes that have both `storage=ssd` and `env=prod`.
- If no node has both labels, the pod stays in Pending state.
----
###### Node Affinity Operators [[K8s VarJosh & Me/Day 17/Deep Dive#Node Affinity Operators (Simple Explanation)|Deep Dive]]


| Operator          | Behavior                                              | Example                                                              |
| ----------------- | ----------------------------------------------------- | -------------------------------------------------------------------- |
| `In`              | Node label must be one of these vaules                | `disk In (ssd, hdd)` Run only on nodes with `disk=ssd` or `disk=hdd` |
| `NotIn`           | Node label must NOT be these values                   | `region NotIn (us-east-1` → Don't run on nodes in `us-east-1`        |
| `Exists`          | The label must not exist (value doesn't matter)       | `gpu Exist` → Run on any node that has a `gpu` label.                |
| `DoesNotExist`    | The label must not exit                               | `gpu DoesNotExists` → Run only on nodes without a `gpu` label        |
| Gt (Greater than) | Label value **must be greater than** the given number | `cpu Gt 8` → Run on nodes with CPU label greater than 8.             |
| Lt (Less than)    | Label value **must be less than** the given number    | `version Lt 5` → Run on nodes with version label less than 5.        |
|                   |                                                       |                                                                      |
###### Example: Using All Operators
```
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage
          operator: In
          values:
            - ssd
            - hdd
        - key: env
          operator: NotIn
          values:
            - dev
        - key: high-memory
          operator: Exists
        - key: dedicated
          operator: DoesNotExist
        - key: cpu
          operator: Gt
          values:
            - "4"
        - key: disk
          operator: Lt
          values:
            - "500"
```

###### Explanation

| Condition                | Effect                                                                    |
| ------------------------ | ------------------------------------------------------------------------- |
| `storage In (ssd, hdd)`  | ✅ Pod can be scheduled if the node has `storage=ssd OR storage=hdd`.      |
| `env NotIn (dev)`        | ✅ Pod is not scheduled on nodes labeled `env=dev`                         |
| `high-memory Exist`      | ✅ Pod can be scheduled **only on nodes that have a** `high-memory` label. |
| `dedicated DoesNotExist` | ✅ Pod avoid nodes with `dedicated` label.                                 |
| `CPU Gt 4`               | ✅ Pod can be scheduled **only on nodes with CPU > 4**.                    |
| `disk Lt 500`            | ✅ Pod can be scheduled **only on nodes with disk < 500**                  |

---
###### Node Anti-Affinity

- `NotIn` and `DoesNotExist` can **prevent pods from running on specific nodes**.
- **Alternative:** Use **taint and tolerations** to **repel/prevent pods** from certain nodes. 
---
###### Summary

✔️ Node Affinity is more flexible than `nodeSelector`.
✔️ Supports **AND/OR** conditions using `matchExpressions`.
✔️Allows both strict (`requiredDuringSchedulingIgnoreDuringExecution`) and (`preferredDuringSchedulingIgnoreDuringExecution`) rules.
✔️ Use `NotIn` and `DoesNotExist` for anti-affinity behavior.

---
###### Understanding`preferredDuringSchedulingIgnoreDuringExecution`
means:

> **"Try to place the Pod on this node, but it's okay if you can't."**

- **preferred** → It's a **preference**, not a strict rule.
    
- **DuringScheduling** → Kubernetes checks this **when scheduling the Pod**.
    
- **IgnoredDuringExecution** → After the Pod is running, Kubernetes **won't move it** even if the preference is no longer met.
    

### Simple example

> "I **prefer** a window seat, but if it's unavailable, I'll take any seat."

Kubernetes thinks the same way:

- If the preferred node is available → ✅ Place the Pod there.
    
- If not → ✅ Place the Pod on another suitable node.
##### How Node Affinity Weight Works

**Weight** tells Kubernetes **how much a node is preferred**.

- **Higher weight** = Higher priority.
    
- **Lower weight** = Lower priority.
    

### Example

- Node A → Weight **100** ⭐⭐⭐⭐⭐
    
- Node B → Weight **50** ⭐⭐⭐
    
- Node C → Weight **10** ⭐
    

Kubernetes will **try Node A first** because it has the highest weight.

### Easy memory trick

> **Weight = Preference Score**

**Higher score → More preferred.**

-----
###### Demonstration: Using Preferred Node Affinity

Step 1: Apply Labels to Nodes
We need to ensure our worker nodes have labels for `storage type`.
```
kubectl label nodes my-first-cluster-worker storage=ssd
kubectl label nodes my-first-cluster-worker2 storage=hdd
```
---

Step 2: Define Node Affinity with Preferences
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: preferred-na-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage
                operator: In
                values:
                  - ssd
                  - hdd
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 10
              preference:
                matchExpressions:
                  - key: storage
                    operator: In
                    values:
                      - ssd
            - weight: 5
              preference:
                matchExpressions:
                  - key: storage
                    operator: In
                    values:
                      - hdd
      containers:
        - name: nginx
          image: nginx
```

###### Explanation

- The Pod is required to run on nodes labeled `storage=ssd` `OR` `storage=hdd` (hard rule).
- The **scheduler prefers** `storage=ssd` nodes (higher weight: `10`).
- The **scheduler prefers** `storage=hdd` nodes slightly less (lower weight: `5`)
----
Step 3: Deploy and Observe Pod Placement
```
kubectl apply -f preferred-na-deploy.yaml
kubectl get pods -o wide
```

###### Expected Behavior

- Pods **prefer** `my-second-cluster-worker` (`storage=ssd`) **over** `my-second-cluster-worker2` (`storage=hdd`)
- If there are not enough resources on `my-second-cluster-worker`, some pods may be placed on `my-second-cluster-worker2`.
- Increasing **replica count** makes the preference **more noticeable.

---
###### Key Considerations

- Weight only influences placements; it does not guarantee it.
- Pods may still be placed on lower-weight nodes if the preferred nodes lack resources.
- Other scheduling factors **still apply**, such as:
	- Available CPU & memory on each node.
	- Other affinity/anti-affinity rules.
	- Taints & Tolerations.
	
📌 Use `requiredDuringSchedulingIgnoreDuringExecution` for strict placement rules, `preferredDuringSchedulingIgnoreDuringExecution` for weighted preferences.

------
###### Node Affinity, `nodeSeletor`, and Taints & Tolerations

Node affinity (`nodeSelector, preferredDuringSchedulingIgnoreDuringExection)

- `nodeSelector` is not retroactive-- it only applies during scheduling.
- `preferredDuringSchedulingIgnoreDuringExecution` means that once a pod is scheduled, it won't be re-evaluated/checked if the node's labels change.
- **Node affinity behaves the same way** -- If a node's labels change after scheduling, pods will not be evicted or rescheduled automatically.
----
###### Behavior of `nodeSelector`

`nodeSelector` is **not retroactive**. If you modify a node's labels after a pod has already been scheduled, the pod **will not be evicted or rescheduled** to a different node.

1. **At Scheduling Time:** The Scheduler assigns the pod to a node based on the labels present at the time.
2. **After Scheduling:** If the node's labels change later, the pod **remains on the same node** unless manually deleted and recreated.

###### Taints & Tolerations (`NoExecute` Effect)

- The `NoExecute` taint effect supports re-evaluation/review/second look —if a node's taints change and a pod doesn't tolerate them, it will be evicted.
-----
## References

[](https://github.com/CloudWithVarJosh/CKA-Certification-Course-2025/tree/main/Day%2017#references)

- [Kubernetes Official Documentation: Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Understanding Node Affinity in Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)
- [Kubernetes Scheduling Preferences](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#types-of-node-affinity)
- [Taints and Tolerations in Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)